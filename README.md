## Automating API Testing with AWS Code Pipeline, AWS Code Build, and Postman

This post demonstrates the use of AWS CodeBuild, CodePipeline and Postman to deploy, 
functionally test an API, and view the generated reports using a new feature from 
CodeBuild called Reports. 

Please refer to the following blog post for instructions: https://aws.amazon.com/blogs/devops/automating-your-api-testing-with-aws-codebuild-aws-codepipeline-and-postman/



## License

This library is licensed under the MIT-0 License. See the LICENSE file.

# Implementazione di un sistema di CI con AWS Pipeline, AWS CodeBuild e AWS CodeDeploy
 
### Aggiunta dei file utili
I file da aggiungere al repository sono:
- **pipeline.yml**: utilizzato da cloudformation per l'istanziamento della pipeline
- **buildspec.yml**: letto da CodeBuild per la build del progetto
- **newman-buildspec.ym**: per l'esecuzione dei test con `newman`,
Struttura progetto risultante:
```bash
|_ postman
    |_ <POSTMAN_COLLECTION>.json (viene aggiunto durante la build)
    |_ <POSTMAN_ENV_COLLECTION>.json (viene aggiunto durante la build)
    |_ update-postman-env-file.sh
|_ template.yml
|_ pipeline.yml
|_ buildspec.yml
|_ newman-buidlspec.yml
```

### Modifica del file buildspec
Specificare nel file `buildspec.yml` il nome del bucket, il nome del template e quello del template di output. Non specificando la cartella di destinazione il file viene messo nella directory principale. 
```yml
version: 0.2
phases:
  install:
    commands:
      - cd 01api
      - aws cloudformation package --template-file <REPLACE_ME_WITH_TEMPLATE_NAME>.yaml
                                   --s3-bucket <REPLACE_ME_WITH_UNIQUE_BUCKET_NAME>
                                   --s3-prefix api-code
                                   --output-template-file <REPLACE_ME_WITH_OUTPUT_TEMPLATE_NAME>.yaml
artifacts:
  type: zip
  files:
    - <REPLACE_ME_WITH_OUTPUT_TEMPLATE_NAME>.yaml
```
modifica del file newman-buildspec.yml:
```yml
version: 0.2

env:
  variables:
    key: "S3_BUCKET"

phases:
  install:
    runtime-versions:
      nodejs: 10
    commands:
      - npm install -g newman
      - yum install -y jq

  pre_build:
    commands:
      - aws s3 cp "s3://${S3_BUCKET}/postman-env-files/<REPLACE_ME_WITH_POSTMAN_ENV_NAME>.json" ./postman/
      - aws s3 cp "s3://${S3_BUCKET}/postman-env-files/<REPLACE_ME_WITH_POSTMAN_COLLECTION_NAME>.json" ./postman/
      - cd ./postman
      - ./update-postman-env-file.sh

  build:
    commands:
      - echo Build started on `date` from dir `pwd`
      - newman run <REPLACE_ME_WITH_POSTMAN_COLLECTION_NAME>.json --environment <REPLACE_ME_WITH_POSTMAN_ENV_NAME>.json -r junit

reports:
  JUnitReports: # CodeBuild will create a report group called "SurefireReports".
    files: #Store all of the files
      - '**/*'
    base-directory: '<REPLACE_ME_WITH_LOCATION_OF_REPORTS>/newman' # Location of the reports
```
### Salvare su S3, sullo stesso bucket, il JSON della collezione da testare e il relativo JSON di environment
```bash
aws s3 cp PetStoreAPI.postman_collection.json \
s3://<REPLACE_ME_WITH_UNIQUE_BUCKET_NAME>/postman-env-files/<REPLACE_ME_WITH_JSON_COLLECTION_NAME>.json
```

```bash
aws s3 cp <REPLACE_ME_WITH_JSON_POSTMAN_ENVIRONMENT_NAME>.json \
s3://<REPLACE_ME_WITH_UNIQUE_BUCKET_NAME>/postman-env-files/<REPLACE_ME_WITH_JSON_POSTMAN_ENVIRONMENT_NAME>.json
```

nel mio caso per eseguire `aws` devo scrivere per intero `/usr/local/bin/aws`.
Quindi ogni comando diventa:

```bash
/usr/local/bin/aws s3 cp PetStoreAPI.postman_collection.json \
s3://<REPLACE_ME_WITH_UNIQUE_BUCKET_NAME>/postman-env-files/<REPLACE_ME_WITH_JSON_COLLECTION_NAME>.json
```
e

```bash
/usr/local/bin/aws s3 cp <REPLACE_ME_WITH_JSON_POSTMAN_ENVIRONMENT_NAME>.json \
s3://<REPLACE_ME_WITH_UNIQUE_BUCKET_NAME>/postman-env-files/<REPLACE_ME_WITH_JSON_POSTMAN_ENVIRONMENT_NAME>.json
```

### AWS CLI to Deploy AWS CloudFormation template

```bash
/usr/local/bin/aws cloudformation create-stack --stack-name petstore-api-pipeline \
--template-body file://./petstore-api-pipeline.yaml \
--parameters \
ParameterKey=BucketRoot,ParameterValue=<REPLACE_ME_WITH_UNIQUE_BUCKET_NAME> \
ParameterKey=GitHubBranch,ParameterValue=<REPLACE_ME_GITHUB_BRANCH> \
ParameterKey=GitHubRepositoryName,ParameterValue=<REPLACE_ME_GITHUB_REPO> \
ParameterKey=GitHubToken,ParameterValue=<REPLACE_ME_GITHUB_TOKEN> \
ParameterKey=GitHubUser,ParameterValue=<REPLACE_ME_GITHUB_USERNAME> \
--capabilities CAPABILITY_NAMED_IAM
```
per generare il token di Github: [link](https://github.com/settings/tokens/new)

### CodeBuild'test reporting
Dalla pagina della pipeline è possibile vedere le fasi, e su CodeBuild viene generato un report riasuntivo dei test eseguiti.
In "Visualizza gruppo di report" vengono riassunti i dati relative a tutte le "run" eseguite.

--capabilities CAPABILITY_NAMED_IAM
