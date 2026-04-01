---
package-name: Fuuga
package-logo: https://bitbucket.org/Thorium/fuuga/raw/72ff92be537745af16919252e6a4b01534ee8b88/docs/logo.png
package-nuget-link: https://www.nuget.org/packages/Fuuga/
package-github-link: https://bitbucket.org/Thorium/fuuga/
package-documentation-link: https://bitbucket.org/Thorium/fuuga/
package-description: Large Language Model (LLM) generator. And Small as well, SML. Give data and generate your model. Fuuga implements a complete language model pipeline: tokenization, data ingestion, model training, fine-tuning, and text generation. Code in .NET (F#).
package-tags: LLM,large,language,model,generator,SML,small,Fuuga,tokenization,data,ingestion,model,training
---

Fuuga is an LLM (Large Language Model) built from scratch in F# and .NET. It implements a complete language model pipeline: tokenization, data ingestion, model training, fine-tuning, and text generation -- with no Python dependencies. Built on TorchSharp for tensor operations and Microsoft.ML.Tokenizers for BPE, Fuuga uses idiomatic F# throughout and works on GPU or CPU. ConfigWizard automatically detects your hardware (GPU VRAM, system RAM) and produces Chinchilla-optimal model and training configurations.

Here is the full pipeline -- tokenize, ingest, train, and generate -- based on the [complete-pipeline.fsx](https://bitbucket.org/Thorium/fuuga/src/master/examples/complete-pipeline.fsx) example:

```fsharp

// This is the Nvidia CUDA package for using GPU:
// #r "nuget: Fuuga"

// This is the CPU package to use only CPU instead:
#r "nuget: Fuuga.cpu"

#r "nuget: TorchSharp"

open System.IO
open Fuuga.Types
open Fuuga.Tokenizer
open Fuuga.Ingest
open Fuuga.Model
open Fuuga.Checkpoint
open Fuuga.Training
open Fuuga.Inference
open Fuuga.Tensor
open Fuuga.ConfigWizard

let dataDir = "data/raw"           // directory containing .txt or .epub files
let tokenizerDir = "data/tokenizer"
let checkpointDir = "checkpoints"

// 1. Train a BPE tokenizer on your corpus
let docs = discoverDocuments dataDir (Some defaultContentFilter)
let corpus = docs |> List.map (fun d -> d.Text)
let corpusBytes = corpus |> List.sumBy (fun t -> int64 (System.Text.Encoding.UTF8.GetByteCount t))

let tokConfig: TokenizerConfig = {
    VocabSize = recommendTokenizerVocabSize corpusBytes
    SpecialTokenRange = (512, 576)
    OutputDir = tokenizerDir
}
trainBpe tokConfig corpus
let tokenizer = loadTokenizer tokenizerDir

// 2. Ingest and tokenize documents into a binary corpus
let corpusPath = "data/corpus.bin"
let tokenizedCorpus = writeCorpus tokenizer docs corpusPath

// 3. ConfigWizard: auto-detect hardware and produce optimal configs
let hw = detectGpu ()
let wizardResult = generateConfigs {
    TotalTokens = tokenizedCorpus.TokenCount
    SampleCount = docs.Length
    AvgDocumentLength = int (tokenizedCorpus.TokenCount / int64 (max 1 docs.Length))
    Hardware = hw
    Goal = Balanced
    MaxTrainingHours = ValueNone
    ExistingVocabSize = ValueSome (embeddingVocabSize tokenizer)
    NvmeSwapPath = None
}

// 4. Train the model
let device = selectDevice ()
let model = new FuugaModel("fuuga", wizardResult.ModelConfig)
train model wizardResult.ModelConfig wizardResult.TrainingConfig
     corpusPath tokenizerDir checkpointDir device

// 5. Generate text from the best checkpoint
match findBestCheckpoint checkpointDir with
| Some checkpoint ->
    let inferModel = new FuugaModel("fuuga", wizardResult.ModelConfig)
    loadCheckpoint checkpoint inferModel None device |> ignore
    (inferModel :> TorchSharp.torch.nn.Module).eval() |> ignore

    let genConfig = {
        GenerationConfigs.defaultGeneration with
            MaxTokens = 64; Temperature = 0.8f
            TopK = 50; TopP = 0.95f; RepetitionPenalty = 1.1f
    }
    let input: InferenceInput = { Text = "Once upon a time"; Prefix = None; Suffix = None }
    let result = generate inferModel tokenizer genConfig input device
    // Expect nonsense when too small corpus and training data!
    printfn "%s" result.Text
| None -> printfn "No checkpoint found."
```
