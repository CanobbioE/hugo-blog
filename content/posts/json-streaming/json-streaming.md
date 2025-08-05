---
title: "Handling huge JSON files"
date: 2025-08-04T15:27:21+02:00
showDate: false
draft: false
tags: ["blog","golang","temporal","back end"]
---

## Introduction
When working with data pipelines, it's easy to underestimate how quickly memory usage can spiral out of control. This post goes over a real-world issue we hit while handling large JSON files in a Temporal activity. The initial implementation worked fine with small test data, but completely fell apart when scaled to production-sized inputs. Here's how we fixed it using JSON streaming, concurrency, and a few practical shortcuts.


## The situation

I was working on a data processing pipeline using a [Temporal Workflow](https://temporal.io/).  
One of the [activities](https://docs.temporal.io/activities) was responsible for receiving a list of links to Google Cloud Storage. Each link pointed to a JSON file containing the results of previous activities.

The activity job was pretty straightforward:

- download data from GCS  
- parse the results  
- send data to a microservice to be stored in a DB  

The first proof-of-concept implementation was as simple as you can imagine:

```go
	for _, reportURL := range reports {
		rawBytes, err := storageClient.Read(ctx, reportURL)
		// check and handle Read error

		var report Report
		err = json.Unmarshal(rawBytes, &report)
		// check and handle Unmarshal error
		// create a batch to send to the dbService.Store()
	}
```

## The issue

This implementation worked well when the reports were just a few MB of test data.  
When it faced the first "real" test dataset, the flaws were obvious: memory usage skyrocketed to 5GB, the database was swarmed by write requests and then the whole service died.

To help understand the issue, here's the report structure:

```json
{
  "name": "Example",
  "id": "space-123",
  "errors": [],
  "job_definitions": [
    {
      "name": "job1",
      "coefficient": 0.5
    }
  ],
  "processed_data": [
    {
      "data_id": "001",
      "data_external_id": "ext-001",
      "data_name": "Name A",
      "final_result": 50,
      "job_partial_result": [
        {
          "partial_result": 100,
          "name": "job1",
          "warnings": [],
          "error": null,
          "settings": {
            ...
          }
        }
      ]
    }
  ]
}
```

As expected, the `processed_data` field is the culprit: it grows bigger the more data is processed. In a real-life environment, this means tens of millions of records.

Obviously, this wasn’t a solid approach for a production environment.  
Still, it served its purpose for the PoC, showing that the feature could work.

Time to improve it.

## The solution

The first thing we did was identify which data we needed to generate the request for `dbService.Store`.  
The top-level fields `name`, `id`, `errors`, and `job_definitions` are required, which is fine since they don't grow like `processed_data`.  
Speaking of which, we unfortunately need most of the data in `processed_data`, so we can't just ignore it. However, we don’t need the entire `settings` object.

After some research, we landed on using [jstream](https://github.com/bcicen/jstream).  
This library provides a way to parse JSON as a stream at a predetermined level:

```go
// using level 2 to parse inner objects like data_id, data_external_id, etc.
decoder := jstream.NewDecoder(reader, 2)
defer decoder.Close()

// EmitKV emits key-value pairs for each parsed object
for mv := range decoder.EmitKV().Stream() {
    // check and handle decoder errors with decoder.Err()
    // process MetaValue
}
```

To easily handle the output while iterating through `Stream()`, we created a simple wrapper:

```go
type StreamResult jstream.KV

func (sr *StreamResult) GetString() string

func (sr *StreamResult) GetFloat32() float32

func (sr *StreamResult) GetStringArray() []string

func (sr *StreamResult) GetStreamResultArray() [][]*StreamResult

func (sr *StreamResult) GetAsJSON() ([]byte, error)
```

With this, we’re almost set.  
As mentioned, we need the top-level fields `name`, `id`, `errors`, and `job_definitions`, but our decoder is set to read at level `2`, so those fields are skipped.

The workaround is to unmarshal those fields separately in a slimmer version of the full report:

```go
type topLevelReportItems struct {
    Name            string              `json:"name"`
    ID              string              `json:"id"`
    Errors          []string            `json:"errors"`
    JobDefinitions  []*JobDefinition    `json:"job_definitions"`
} 

// ...

var partialReport topLevelReportItems
err = json.NewDecoder(reader).Decode(&partialReport)
// check and handle Decode error
```

One last issue: `storageClient.Read(ctx, reportURL)` loads the entire GCS object into memory.  
GCS objects have a built-in method to avoid this: `object.NewReader()`, which returns a stream reader.

Before putting everything together, we decided to optimize further by parsing each report concurrently.  
Each report’s `processed_data` is parsed asynchronously (producer), and the parsed items are collected and sent in a single batch to the `dbService` via gRPC (consumer).  
The `dbService` implements a transparent proxy to our database.

We’ll skip the proxy implementation for now: it might deserve its own post.  
Here’s the full solution:

```go
	for _, reportURL := range reports {
		obj, err := storageClient.Object(reportURL)
		// check and handle Object error

		topLevelReader, err := obj.NewReader(ctx)
		// check and handle NewReader error
		defer topLevelReader.Close()

		var partialReport topLevelReportItems
		err = json.NewDecoder(reportReader).Decode(&partialReport)
		// check and handle Decode error

		// create a second reader because topLevelReader is exhausted
		innerReader, err := obj.NewReader(ctx)
		// check and handle NewReader error
		defer innerReader.Close()

		decoder := jstream.NewDecoder(innerReader, 2)
		defer decoder.Close()

		wg.Add(1)
		go func() {
			defer wg.Done()
			for mv := range decoder.EmitKV().Stream() {
                // check and handle decoder errors with decoder.Err()
				streamResults := jsonutils.FromMetaValue(mv)
				data := &Data{}
				for _, result := range streamResults {
					switch result.Key {
					case "data_id":
						data.ID = result.GetString()
                    // ... handle other fields ...
					case "job_partial_result":
						jobResult := &JobResult{}
						for _, jobResults := range result.GetStreamResultArray() {
							for _, jr := range jobResults {
								switch jr.Key {
								case "partial_result":
									jobResult.PartialResult = jr.GetFloat32()
								case "name":
									jobResult.Name = jr.GetString()
								case "warnings":
									jobResult.Warnings = jr.GetString()
								case "error":
									jobResult.Error = jr.GetString()
								}
								// skip `settings` since it's unused
							}
						}
						data.JobResults = append(data.JobResults, jobResult)
					}
				}
				// send data to consumer, to be persisted
				dataChan <- data
			}
		}()
	}
```

## The result

The final version is a bit less elegant than the initial prototype, but it’s a huge win in terms of memory efficiency.  
To measure the improvement, we added strategic breakpoints in the code to store memory stats from the [runtime package](https://pkg.go.dev/runtime) into a CSV.

After some spreadsheet work, we got a clean graph:

![graph showing memory usage is under 200mb](/posts/json-streaming/chart.png)

The graph shows memory usage building up to around 180MB while the consumer builds the batch, then dropping back below 100MB once the batch is sent and processed.

## In conclusion

Streaming JSON with tools like `jstream` can dramatically reduce memory usage when dealing with large payloads.  
By parsing only what’s needed and handling it incrementally, we avoided loading gigabytes of data into memory all at once.  
This approach turned a crash-prone prototype into a production-ready workflow with consistent and predictable resource usage.
