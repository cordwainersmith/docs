# aces-bvr-transcription-api

## Transcribe Endpoint

The Transcribe Endpoint is a crucial part of our audio processing service, designed to convert audio recordings into written transcripts. This document provides a comprehensive overview of the endpoint, including its functionality, implementation details, and how to use it.

### Endpoint Description

`POST /transcribe`

This endpoint accepts an audio file and transcription settings, processes the audio to generate a transcript, and returns the results in a structured format.

### Request Parameters

- `request`: A `TranscriptionRequest` object containing:
  - `transcription_mode`: Specifies the mode of transcription. Valid options are "single" for individual files or "batch" for processing multiple files.
  - `uri`: The URI of the audio file to be transcribed.
- `api_key`: Authentication API key required for accessing the endpoint. This is validated through a dependency, `get_api_key`.

### Response Format

The endpoint returns a JSON object with the following key:
- `Results`: Contains the transcription results. The structure of this object depends on the transcription mode and the content of the audio file.

### Error Handling

If an error occurs during the processing, the endpoint raises an `HTTPException` with status code 500 and a detailed error message.

### Implementation Details

#### Overview

The transcribe endpoint leverages asynchronous Python features (`asyncio`) to offload the transcription task to a separate thread, ensuring non-blocking behavior of the web service.

#### Core Components

- **Transcription Service**: The core logic for audio transcription is encapsulated in the `/utils/transcript_service.py` module. This includes audio file management, interaction with Azure Cognitive Services for speech-to-text conversion, and post-processing of transcription results.

- **Swagger Client Configuration**: Uses a swagger client to interact with Azure's Speech to Text API, configured with subscription keys and service regions from environment variables.

- **Transcription Properties**: Defines properties for the transcription request, such as diarization, punctuation mode, and profanity filter settings.

- **Audio Processing**: Includes downloading the audio file, converting it to mono (if necessary), uploading it to Azure Blob Storage, and generating a SAS token for secure access.

- **Database Interaction**: Utilizes data access objects (DAOs) for storing and retrieving transcription results and reports from the database.

### Using the Endpoint

1. **Authentication**: Ensure you have a valid API key for authentication.
2. **Prepare Request**: Formulate a `TranscriptionRequest` with the appropriate `transcription_mode` and the audio file's `uri`.
3. **Call Endpoint**: Use an HTTP client to `POST` the request to `/transcribe` along with the `api_key`.
4. **Handle Response**: Process the JSON response, which includes the transcription results or an error message if applicable.

### Error Codes

- `500 Internal Server Error`: Indicates an unexpected error occurred during the transcription process. Check the error message for more details.

### Sequence Diagram
```mermaid
sequenceDiagram
    participant Client
    participant TranscribeEndpoint as Transcribe Endpoint
    participant APIKeyValidation as API Key Validation
    participant TranscribeFunc as Transcribe Function
    participant DataAccess as Data Access Layer
    participant AzureBlob as Azure Blob Storage
    participant TranscriptionClient as Transcription Client
    participant AnalysisService as Analysis Service

    Client->>+TranscribeEndpoint: POST /transcribe (request)
    TranscribeEndpoint->>+APIKeyValidation: Validate API Key
    APIKeyValidation-->>-TranscribeEndpoint: Validation Success
    TranscribeEndpoint->>+TranscribeFunc: transcribe(request.transcription_mode, request.uri)
    TranscribeFunc->>+DataAccess: Check if transcription exists
    DataAccess-->>-TranscribeFunc: Transcription does not exist
    TranscribeFunc->>+AzureBlob: Upload audio to Blob
    AzureBlob-->>-TranscribeFunc: Blob URI
    TranscribeFunc->>+TranscriptionClient: Start Transcription
    TranscriptionClient->>+AzureBlob: Fetch audio
    AzureBlob-->>-TranscriptionClient: Audio stream
    TranscriptionClient-->>-TranscribeFunc: Transcription ID
    loop Check Status
        TranscribeFunc->>TranscriptionClient: Check Transcription Status
        TranscriptionClient-->>TranscribeFunc: Status (Pending)
    end
    TranscriptionClient-->>TranscribeFunc: Status (Succeeded)
    TranscribeFunc->>TranscriptionClient: Download Transcription Files
    TranscriptionClient->>+DataAccess: Save Transcription Results
    DataAccess-->>-TranscribeFunc: Save Success
    opt Analysis Enabled
        TranscribeFunc->>+AnalysisService: Run Analysis
        AnalysisService-->>-TranscribeFunc: Analysis Results
    end
    TranscribeFunc-->>-TranscribeEndpoint: Transcription Completed
    TranscribeEndpoint-->>-Client: Return Results

```
### Flowchart Diagram
```mermaid
flowchart TB
    start((Start)) --> transcribe_endpoint[/transcribe_endpoint/]
    transcribe_endpoint -->|asyncio.to_thread| transcribe[transcribe]
    transcribe -->|extract_engagement_id_from_uri| check_engagement_id{Engagement ID Valid?}
    check_engagement_id -->|No| return_invalid_url{{"Invalid URL format"}}
    check_engagement_id -->|Yes| check_transcription_exists{Transcription Exists?}
    check_transcription_exists -->|Yes| return_exists{{"Transcription already exists"}}
    check_transcription_exists -->|No| download_and_convert_to_mono[download_and_convert_to_mono]
    download_and_convert_to_mono --> upload_to_azure_blob_path[upload_to_azure_blob_path]
    upload_to_azure_blob_path --> generate_blob_sas_url[generate_blob_sas_url]
    generate_blob_sas_url --> configure_client[configure_client]
    configure_client --> init_transcription_properties[init_transcription_properties]
    init_transcription_properties -->|transcription_mode| transcription_mode_decision{Transcription Mode}
    transcription_mode_decision -->|single| transcribe_from_single_blob[transcribe_from_single_blob]
    transcription_mode_decision -->|batch| transcribe_from_container[transcribe_from_container]
    transcription_mode_decision -->|Invalid| raise_error{{"Invalid transcription mode"}}
    transcribe_from_single_blob --> create_transcription[create_transcription]
    transcribe_from_container --> create_transcription
    create_transcription --> check_transcription_status[check_transcription_status]
    check_transcription_status -->|status| status_decision{Status}
    status_decision -->|Failed| log_error[log_error]
    status_decision -->|Succeeded| download_and_process_transcription_files[download_and_process_transcription_files]
    download_and_process_transcription_files --> process_transcription_file[process_transcription_file]
    process_transcription_file -->|process| process_words_and_speaker_mapping[process_words_and_speaker_mapping]
    process_words_and_speaker_mapping --> get_words_speaker_mapping[get_words_speaker_mapping]
    get_words_speaker_mapping --> get_sentences_speaker_mapping[get_sentences_speaker_mapping]
    get_sentences_speaker_mapping --> write_srt[write_srt]
    write_srt -->|AnalysisEnabled?| analysis_decision{Analysis Enabled?}
    analysis_decision -->|No| end1((End))
    analysis_decision -->|Yes| run_conversation_analysis[run_conversation_analysis]
    run_conversation_analysis --> end2((End))
    raise_error --> end3((End))
    return_invalid_url --> end4((End))
    return_exists --> end5((End))
    log_error --> handle_transcription_report[handle_transcription_report]
    handle_transcription_report --> end6((End))
```
## Fetch Records Endpoint

The Fetch Records Endpoint is designed to retrieve conversational records based on engagement IDs. This document provides details about the endpoint's functionality, how it's implemented, and instructions for use.

### Endpoint Description

`POST /fetch-records`

This endpoint accepts a list of engagement IDs and returns the conversational records associated with those IDs.

### Request Parameters

- `request`: An `EngagementIDRequest` object containing:
  - `engagement_ids`: A list of engagement IDs for which records are to be fetched.
- `api_key`: An authentication API key required for accessing the endpoint. This is validated through a dependency, `get_api_key`.

### Response Format

The endpoint returns a list of dictionaries, each representing a conversational record associated with the provided engagement IDs. The structure of these dictionaries is defined by the data model in the conversational analysis database.

### Error Handling

If an error occurs during the process, the endpoint raises an `HTTPException` with status code 500 and a detailed error message.

### Implementation Details

#### Overview

The fetch records endpoint uses the `fetch_records_by_engagment_ids` function from the `utils.db_helpers` module to query the database for records matching the provided engagement IDs.

#### Core Components

- **Database Query**: The `fetch_records_by_engagment_ids` function constructs a regex pattern based on the engagement IDs and queries the `conversational_analysis_merged_collection` for matching documents.

- **Serialization**: Utilizes the `custom_jsonable_encoder` from `utils.helper` to serialize the MongoDB documents into JSON-compatible format.

### Using the Endpoint

1. **Authentication**: Ensure you have a valid API key for authentication.
2. **Prepare Request**: Create an `EngagementIDRequest` object with the list of engagement IDs for which you want to fetch records.
3. **Call Endpoint**: Use an HTTP client to `POST` the request to `/fetch-records` along with the `api_key`.
4. **Handle Response**: Process the response, which contains a list of dictionaries with the conversational records.

### Sequence Diagram
```mermaid
sequenceDiagram
    participant Client
    participant FastAPI
    participant fetch_records_endpoint
    participant fetch_records_by_engagment_ids
    participant DB

    Client->>+FastAPI: POST /fetch-records
    FastAPI->>+fetch_records_endpoint: Calls endpoint with EngagementIDRequest and API Key
    fetch_records_endpoint->>+fetch_records_by_engagment_ids: Calls function with engagement_ids
    loop For each engagement_id
        fetch_records_by_engagment_ids->>+DB: Query DB with regex pattern
        DB-->>-fetch_records_by_engagment_ids: Returns matching documents
    end
    fetch_records_by_engagment_ids-->>-fetch_records_endpoint: Returns list of records
    fetch_records_endpoint->>+FastAPI: Serializes records
    FastAPI-->>-Client: Returns serialized records as response

    alt If an exception occurs
        fetch_records_endpoint->>FastAPI: Raises HTTPException
        FastAPI-->>Client: Returns error response
    end
```
### Flowchart Diagram
```mermaid
flowchart TB
    A[Start: fetch_records_endpoint] --> B{Is API Key Valid?}
    B -- Yes --> C[Fetch records by engagement IDs]
    B -- No --> Z[Return HTTPException: Unauthorized]
    C --> D{Did fetch_records_by_engagment_ids succeed?}
    D -- Yes --> E[Serialize records]
    E --> F[Return serialized records]
    D -- No --> G[Log Exception]
    G --> H[Return HTTPException: Internal Server Error]

    C --> I[Start: fetch_records_by_engagment_ids]
    I --> J[For each engagement ID]
    J --> K[Construct regex pattern]
    K --> L[Query conversational_analysis_merged_collection]
    L --> M{Did find matching documents?}
    M -- Yes --> N[Append documents to records]
    N --> O[Return records to fetch_records_endpoint]
    M -- No --> P[Continue to next engagement ID]    
```
### Error Codes

- `500 Internal Server Error`: Indicates an unexpected error occurred during the fetching process. Check the error message for more details.



### Development and Testing

- **Environment Setup**: Ensure all required environment variables are set, including `SUBSCRIPTION_KEY`, `SERVICE_REGION`, and Azure Blob Storage settings.
- **Dependencies**: Install all necessary Python packages as listed in `requirements.txt`.


