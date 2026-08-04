
```mermaid

flowchart TD

    %% Connections
    STEP0 --> STEP1
    STEP1 --> db_cortex1[(cortex_youtube_demo)]
    STEP1 --> STEP2

    STEP2 --> db_topics[(mart_cortex_topics)]
    STEP2 --> doc_video[/2_2_aggregation_query/]
    STEP2 --> STEP2B

    STEP2B --> db_exp[(2B_4_mart_cortex_topic_experiment)]
    STEP2B --> doc_temp[/2B_6_final_deterministic_aggregation/]
    STEP2B --> STEP2C

    STEP2C --> db_cat[(2C_1_mart_cortex_topics)]
    STEP2C --> doc_cat[/2C_2_ground_truth/]
    
```
