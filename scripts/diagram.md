
```mermaid

flowchart TD

    %% Connections
    STEP0 --> STEP1
    STEP1 --> db_cortex1[(cortex_youtube_demo)]
    STEP1 --> STEP2

    STEP2 --> db_topics[(mart_cortex_topics)]
    STEP2 --> doc_video[/2_2_aggregation_query/]
    STEP2 --> STEP3

    STEP3 --> STEP2B
    STEP2B --> db_exp[(mart_cortex_topic_experiment)]
    STEP2B --> doc_temp[/temp/]
    STEP2B --> STEP2B4
    
```
