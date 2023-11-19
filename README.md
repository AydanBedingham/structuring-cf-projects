# Structuring CloudFormation Projects

This project explores various approaches to structuring CloudFormation Projects.

The architectures examined is a simple ingestion pipeline:
- Defines S3 buckets for Incoming JSON and Incoming CSV files.
- Each bucket has a respective SQS queue which will receive items in response to files being uploaded to the bucket (s3:ObjectCreated:*).
- Lambda functions consume items from the SQS queues, process the files and transfer them to a seperate Curated JSON bucket.
    * JSON files are copied from the Incoming JSON bucket to the Curated JSON bucket.
    * CSV files are read from the Incoming CSV bucket, converted to JSON and transferred to the Curated JSON bucket.
    * SQS items are processed in groups with item processing failures handled using partial batch responses. Repeated failures will reuslt in the SQS item being transferred to a corresponding Dead Letter Queue (DLQ).


![Screenshot](images/simple-ingestion-pipeline.svg?raw=true)


## Approaches

|                | Reduced CF Parameters | Avoids Resource Naming Constraints | Modular Template Files  | Nested Stack  | Reduced CF Exports |
| -:             | :-:                   | :-:                                | :-:                     | :-:           | :-:                |
| **Approach 1** | &cross;               | &cross;                            | &cross;                 | &cross;       | &cross;            |
| **Approach 2** | &check;               | &cross;                            | &cross;                 | &cross;       | &cross;            |
| **Approach 3** | &check;               | &check;                            | &cross;                 | &cross;       | &cross;            |
| **Approach 4** | &check;               | &check;                            | &check;                 | &cross;       | &cross;            |
| **Approach 5** | &check;               | &check;                            | &check;                 | &check;       | &cross;            |
| **Approach 6** | &check;               | &check;                            | &check;                 | &check;       | &check;            |

### Approach 1: Single template file, all resources named via parameters.
- Entire CloudFormation stack is defined in a single template file ie. `ingestion-pipeline.yml`.
- Names of every resource are defined individually using Parameters.
- Using Parameters to define resource names can be used to ensure uniqueness but is very tedious to fill-out during stack deployment.

### Approach 2: Static resource names with StackName prefix to ensure resource uniqueness.
- StackName used as a resource name prefix.
- Using the StackName to prefix resource names ensures uniqueness and is less tedious than Parameters.
- Incorporating the StackName as a resource name prefix means descriptive stack names could risk violating resource name length constraints.

### Approach 3: Static resource names with prefix parameter used to mitigating resource naming constraints.
- Prefix parameter used as resource name prefix instead of StackName parameter.
- Using a Prefix parameter instead of the StackName allows for descriptive stack names without violating resource name length constraints.

### Approach 4: Multiple template files to increase readability.
- Resources are spread across multiple CloudFormation stacks defined across multiple template files. [See Here](images/simple-ingestion-pipeline-grouped.svg?raw=true "Diagram grouped by template file")
    * `bucket-incoming-json.yml` : Bucket and Queues for incoming json.
    * `bucket-incoming-csv.yml` : Bucket and Queues for incoming csv.
    * `bucket-curated-json.yml`: Bucket and Queues for curated json.
    * `lambda-copy-incoming-json-to-curated-json.yml`: Lambda to copy incoming json to curated json bucket.
    * `lambda-copy-incoming-csv-to-curated-json.yml`: Lambda to copy incoming csv to curated json bucket. 
- Values are shared between stacks via exported stack outputs.
- Using multiple templates greatly improved code readability.
- Multiple template files complicates deployment as there are now many more stacks to deploy.
- Usage of stack outputs to share values between stacks complicated deployment by introducing dependencies between stacks (eg. `lambda-copy-incoming-json-to-curated-json.yml` can only be deployed after `bucket-incoming-json.yml` as it is dependent on it's stack outputs).

### Approach 5: Nested stack used to make deployment easier.
- Nested stacks used to reduce deployable templates to a single file containing the parent stack ie. `ingestion-pipeline.yml`.
- DependsOn relationships between nested stacks used to resolve deployment ordering issues.
- The CloudFormation CLI (or SAM CLI) needed to be used to package and deploy the nested stack template. eg. `sam deploy --region us-east-1 --template-file ingestion-pipeline.yml --stack-name ingest-pipeline --parameter-overrides Prefix=ingest --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM  --profile=dev`

### Approach 6: Stack exports reduced by reading outputs directly from nested stacks.
- Parent stack modified to obtain shared values from nested stacks outputs and provide them to other nested  stacks using Parameters.
- Using the parent stack to share values between stacks alleviated the need to export variables from stacks.
