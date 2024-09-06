# AWS Lambda-based service

This code sample defines an AWS Lambda-based service for pulling data from Facebook at regular intervals using the Serverless framework.

## serverless.yml

```yaml
service: data-pull

useDotenv: true

plugins:
  - serverless-python-requirements
  - serverless-offline

provider:
  name: aws
  stackTags:
    service: ${self:service}
  stage: ${opt:stage, 'dev'}
  runtime: python3.9
  region: ${opt:region, 'us-east-1'}
  environment:
    SERVICE_NAME: ${self:service}
    ENV: ${self:provider.stage}
    AWS_ACCOUNT: ${aws:accountId}
    POSTGRES_HOST: ${env:POSTGRES_HOST}
    POSTGRES_USER: ${env:POSTGRES_USER}
    POSTGRES_PASSWORD: ${env:POSTGRES_PASSWORD}
    POSTGRES_DATABASE: ${env:POSTGRES_DATABASE}
    FACEBOOK_APP_ID: ${env:FACEBOOK_APP_ID}
    FACEBOOK_APP_SECRET: ${env:FACEBOOK_APP_SECRET}
    FACEBOOK_ACCESS_TOKEN: ${env:FACEBOOK_ACCESS_TOKEN}
  apiGateway:
    apiKeySourceType: HEADER

package:
  exclude:
    - '*'
    - '!src/**'
    - '**/*.pyc'
    - 'node_modules/**'
    - 'venv/**'
    - '.idea/**'
    - '.github/**'

custom:
  pythonRequirements:
    dockerizePip: false
    usePipenv: true

functions:
  - ${file(src/facebook/facebook-data-pull.yml)}
```

## facebook-data-pull.yml

```yaml
facebook_hourly_data_pull:
  handler: src.facebook.handlers.hourly_fb_data_update
  memorySize: 1024
  timeout: 900
  events:
    - schedule: rate(10 minutes)

facebook_weekly_data_pull:
  handler: src.facebook.handlers.weekly_fb_data_pull
  memorySize: 1024
  timeout: 900
```
