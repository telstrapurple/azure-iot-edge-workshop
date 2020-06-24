# Cognitive Services

![https://unsplash.com/photos/BpTMNN9JSmQ](../.gitbook/assets/image%20%28149%29.png)

We had enough of simulated temperature sensor module! We have seen Stream Analytics in previous exercises. Azure Blob Storage, SQL Server, SQL Database Edge and Azure ML models can be also used. Custom vision exported ONNX models are a good candidate as well. For simplicity sake, let's try Cognitive Services. [Many cognitive services](https://docs.microsoft.com/en-us/azure/cognitive-services/cognitive-services-container-support?tabs=luis) can be used in containers so that works pretty well for IoT Edge. We don't need to do any custom ML training but simply consume the APIs hosted inside the containers directly.

### Create text analytics resource

One of the simplest cognitive services to use is text analytics. It has features like sentiment analysis, key phrase extraction and language detection which is the API used in this tutorial.

Create a text analytics resource and grab the endpoint and API key. Make sure name of text analytics resource is unique and place it in same IoT edge resource group.

```text
cognitiveservices account create --kind TextAnalytics --location southeastasia --sku F0 --resource-group iotedge-rg --name text-analytics-edge01
cognitiveservices account show --resource-group iotedge-rg  --name text-analytics-edge01
cognitiveservices account keys list --resource-group iotedge-rg --name text-analytics-edge01
```

 From the above commands, you can grab the endpoint URL and API key which we would use later.

### Create Edge Solution

Fire up VS Code and create a new edge solution and make sure the target platform is **amd64**. When prompted to select module template, pick the option to use Azure Marketplace. Search for language detection module and import it.

![](../.gitbook/assets/image%20%28196%29.png)

Open deployment template file.

![](../.gitbook/assets/image%20%28172%29.png)

Port 5000 is the port used to receive API calls from other modules. Fill your API key and endpoint address. This information is used for billing. In our case, text analytics resource is created using the free tier which is very enough for our use case. From command palette, build and run IoT edge solution in simulator.

 Terminal window will show logs emitted from language detection module.

![](../.gitbook/assets/image%20%28168%29.png)

Hit [http://localhost:5000](http://localhost:5000) in your browser.

![](../.gitbook/assets/image%20%28102%29.png)

**Service API Description** link is a pointer to a Swagger page with API documentation and option to try the API and make sure things work.

For a POST request like:

```javascript
{
  "documents": [    
    {
      "id": "1",
      "text": "Bonjour tout le monde"
    }
  ]
}
```

We should get this response:

```javascript
{
  "documents": [
    {
      "id": "1",
      "detectedLanguage": {
        "name": "French",
        "iso6391Name": "fr",
        "confidenceScore": 1
      },
      "warnings": []
    }
  ],
  "errors": [],
  "modelVersion": "2019-10-01"
}
```

Cool, module works fine by itself but we want to consume it from another module running on same device. Stop the simulator and proceed to add another custom module to the solution.

### Adding a custom module calling into Text Analytics module

Add another **C\# module** to the solution and name it **ConsumerModule** and update docker image repository to point to your own ACR registry. Also remove all routes from edgeHub routes element, they are not needed.

The plan is to have this C\# module inspect a certain \(host shared\) folder within the file system of the container itself for text files. It will then grab the contents of each text file and submit it to text analytics module.

Modify **createOptions** of **C\# consumer module** to have a bind to the host shared folder. Update the folder path to match your own setup if needed and create the folder.

```javascript
"createOptions": {
  "HostConfig": {
    "Binds":["c:\\temp\\iot-edge:/input-data"]
  }
}
```

Latest versions of docker allow sharing specific folders from host into the container instead of full drives. You can add the folder share in advance like below.

![](../.gitbook/assets/image%20%2847%29.png)

Otherwise, a confirmation message will be shown to create the file share once the container is instantiated.

![](../.gitbook/assets/image%20%2885%29.png)

### Update C\# module to call into Text Analytics module

Replace the contents of C\# module `program.cs` file with the following \(big\) snippet.

```csharp
namespace ConsumerModule
{
    using System;
    using System.IO;
    using System.Linq;
    using System.Net.Http;
    using System.Runtime.Loader;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using Newtonsoft.Json;

    class Program
    {
        static string  inputFolder = "/input-data";

        static async Task Main(string[] args)
        {
            Init();

            // Wait until the app unloads or is cancelled
            var cts = new CancellationTokenSource();
            AssemblyLoadContext.Default.Unloading += (ctx) => cts.Cancel();
            Console.CancelKeyPress += (sender, cpe) => cts.Cancel();

            await WhenCancelled(cts.Token);
        }

        /// <summary>
        /// Handles cleanup operations when app is cancelled or unloads
        /// </summary>
        public static Task WhenCancelled(CancellationToken cancellationToken)
        {
            var tcs = new TaskCompletionSource<bool>();
            cancellationToken.Register(s => ((TaskCompletionSource<bool>)s).SetResult(true), tcs);
            return tcs.Task;
        }

        static void Init()
        {        
            var thread = new Thread(async () => await ProcessDocuments());
            thread.Start();
        }

        private static async Task ProcessDocuments()
        {
            while (true)
            {
                Console.WriteLine($"Inspecting input folder {inputFolder} ....");
                var files = Directory.GetFiles(inputFolder);
                Console.WriteLine($"Found {files.Length} files");
                System.Console.WriteLine();
                try
                {
                    var response = await DetectLanaguage(files);
                    response.ToList().ForEach(x =>
                    {
                        System.Console.WriteLine($"File name : {x.fileName}");
                        System.Console.WriteLine($"File contents : {File.ReadAllText(Path.Combine(inputFolder, x.fileName))}");
                        System.Console.WriteLine($"Detected language : {x.language}");
                        System.Console.WriteLine();
                    });
                    System.Console.WriteLine("".PadRight(80, '#'));
                }
                catch (Exception ex)
                {
                    // probably if text analytics module is not yet ready
                    System.Console.WriteLine(ex.Message);
                }

                // inpsect input folder every 10 seconds
                await Task.Delay(10 * 1000);
            }
        }

        private async static Task<(string fileName, string language)[]> DetectLanaguage(string[] filePaths)
        {
            if (filePaths.Length == 0)
                return Array.Empty<(string fileName, string language)>();

            var documents = filePaths.Select(file =>
            {
                return new RequestDocument
                {
                    Id = Path.GetFileName(file),
                    Text = File.ReadAllText(file)
                };
            }).ToArray();

            var payload = JsonConvert.SerializeObject(new TextAnalyticsRequest {Documents = documents});

            HttpClient client = new HttpClient();
            var response = await client.PostAsync("http://LanguageDetectionContainerAzureCognitiveServices:5000/text/analytics/v3.0/languages?model-version=latest",
                new StringContent(payload, Encoding.UTF8, "application/json"));

            var responseText = await response.Content.ReadAsStringAsync();

            return JsonConvert
                .DeserializeObject<TextAnalyticsResponse>(responseText)
                .Documents
                .Select(x => (x.Id, x.DetectedLanguage.Name))
                .ToArray();
        }
    }

    class RequestDocument
    {
        public string Id { get; set; }
        public string Text { get; set; }

    }
    class TextAnalyticsRequest{
        public RequestDocument[] Documents { get; set; }
    }

    class TextAnalyticsResponse{
        public ResponseDocument[] Documents { get; set; }
    }
    class ResponseDocument
    {
        public string Id { get; set; }

        public DetectedLanguage DetectedLanguage { get; set; }
    }
    public class DetectedLanguage
    {
        public string Name { get; set; }
        public string Iso6391Name { get; set; }
        public float ConfidenceScore { get; set; }
    }
}

```

The main points to highlight in this class are:

* An endless loop doing folder file enumeration and calling a method to detect language every 10 seconds.
* 
Place a few text files with 



From deployment template context menu, select **Build and run** IoT Edge **solution in simulator.**

\*\*\*\*

![](../.gitbook/assets/image%20%2866%29.png)

