---
package-name: FSharp.Azure.Quantum
package-logo: https://raw.githubusercontent.com/Thorium/FSharp.Azure.Quantum/refs/heads/main/docs/images/logo.png
package-nuget-link: https://www.nuget.org/packages/FSharp.Azure.Quantum
package-github-link: https://github.com/Thorium/FSharp.Azure.Quantum
package-documentation-link: https://thorium.github.io/FSharp.Azure.Quantum/
package-description: F# library for Azure Quantum with quantum-first optimization (Graph Coloring, TSP, MaxCut, Knapsack, Network Flow, Portfolio), Azure Quantum backend integration (IonQ, Rigetti, Quantinuum, Atom Computing), extensible circuit validation, local quantum simulation, QAOA with automatic parameter optimization (Nelder-Mead), OpenQASM 2.0 support, error mitigation techniques (ZNE, PEC, REM), Quantum Random Number Generation (QRNG), and educational quantum algorithms (Grover, QFT, Amplitude Amplification) with cloud backend execution. Multi-provider architecture supports IonQ, Rigetti, Quantinuum, Atom Computing, IBM, Amazon Braket, and Google Cirq.
#package-posts-link: optional
package-tags: Quantum,Azure,Computing,Optimization,TSP,Portfolio,QAOA,Circuit
---

FSharp.Azure.Quantum is a quantum-first F# library for solving combinatorial optimization problems using quantum algorithms (QAOA) with automatic cloud/local backend selection. It provides idiomatic F# computation expressions for problem specification, supports multiple quantum backends (LocalBackend, IonQ, Rigetti, Quantinuum, Atom Computing), and includes error mitigation techniques.

Here is an example showing how to solve a portfolio optimization and a knapsack problem using quantum optimization:

```fsharp
#r "nuget: FSharp.Azure.Quantum"

open FSharp.Azure.Quantum

// Use local quantum simulator (swap this with an Azure Quantum backend to run on real quantum hardware)
let backend = Backends.LocalBackend.LocalBackend() :> Core.BackendAbstraction.IQuantumBackend

// Example 1: Portfolio Optimization
let assets = [
    ("AAPL", 0.12, 0.15, 150.0)   // (symbol, return, risk, price)
    ("GOOGL", 0.10, 0.12, 2800.0)
    ("MSFT", 0.11, 0.14, 350.0)
]

let portfolioProblem = Portfolio.createProblem assets 10000.0  // budget

match Portfolio.solve portfolioProblem (Some backend) with
| Ok allocation ->
    printfn "Portfolio value: $%.2f" allocation.TotalValue
    printfn "Expected return: %.2f%%" (allocation.ExpectedReturn * 100.0)
    printfn "Risk: %.2f" allocation.Risk
    allocation.Allocations
    |> List.iter (fun (symbol, shares, value) ->
        printfn "  %s: %.2f shares ($%.2f)" symbol shares value)
| Error msg -> printfn "Error: %A" msg

// Example 2: Knapsack Problem (0/1)
let items = [
    ("laptop", 3.0, 1000.0)   // (id, weight, value)
    ("phone", 0.5, 500.0)
    ("tablet", 1.5, 700.0)
    ("monitor", 2.0, 600.0)
]

let knapsackProblem = Knapsack.createProblem items 5.0  // capacity = 5.0

match Knapsack.solve knapsackProblem (Some backend) with
| Ok solution ->
    printfn "Total value: $%.2f" solution.TotalValue
    printfn "Total weight: %.2f/%.2f" solution.TotalWeight knapsackProblem.Capacity
    printfn "Items: %A" (solution.SelectedItems |> List.map (fun i -> i.Id))
| Error msg -> printfn "Error: %A" msg
```
