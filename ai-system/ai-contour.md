## AI architecture for agent context delivery

```mermaid
flowchart TB

    subgraph PREP["1. Preparation"]
        direction LR

        SC["ScenarioConfig"]
        SH["Shared Context"]
        PC["Player Context"]
        OC["Opponent Private Context"]

        COACH["Preparation Coach - Qwen reasoning ON"]
        PLAN["StrategyPlan"]
        STRAT["OpponentStrategy"]

        SC --> SH
        SC --> PC
        SC --> OC

        SH --> COACH
        PC --> COACH
        COACH --> PLAN

        SH --> STRAT
        OC --> STRAT
    end


    KB["Negotiation Knowledge Base - Embeddings + Qdrant"]

    KB -. "Coach retrieval" .-> COACH
    KB -. "Opponent retrieval" .-> STRAT


    subgraph LIVE["2. Real-time Negotiation"]
        direction LR

        USER["User"]
        STT["Whisper STT"]
        GUARD["Guard - Qwen reasoning OFF"]

        OPP["Opponent - Qwen reasoning OFF"]
        VAL["Response and State Validator"]

        SAFE["Safe In-Role Response"]

        TTS["Qwen TTS"]
        OUT["Opponent Voice"]

        USER --> STT
        STT --> GUARD

        GUARD -->|Allowed| OPP
        GUARD -->|Blocked| SAFE

        OPP --> VAL
        VAL --> TTS
        SAFE --> TTS

        TTS --> OUT
    end


    SH --> OPP
    OC --> OPP
    STRAT --> OPP


    subgraph MEMORY["3. Session State"]
        direction LR

        STATE["SessionState"]
        HIST["Transcript"]

        TA["Turn Analyzer - Qwen reasoning OFF"]
        TAN["TurnAnalysis"]

        STATE --> TA
        HIST --> TA
        TA --> TAN
    end


    VAL --> STATE

    STT --> HIST
    VAL --> HIST
    SAFE --> HIST

    STATE -.-> OPP
    HIST -.-> OPP


    subgraph FINAL["4. Final Evaluation"]
        direction LR

        RUBRIC["Judge Rubric"]
        JUDGE["Final Judge - Qwen reasoning ON"]
        RESULT["JudgeResult"]

        RUBRIC --> JUDGE
        JUDGE --> RESULT
    end