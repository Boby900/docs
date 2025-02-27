---
title: Déploiement sur Amazon Elastic Container Service
intro: Vous pouvez effectuer un déploiement sur ECS (Amazon Elastic Container Service) dans le cadre de vos workflows de déploiement continu (CD).
redirect_from:
  - /actions/guides/deploying-to-amazon-elastic-container-service
  - /actions/deployment/deploying-to-amazon-elastic-container-service
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: tutorial
topics:
  - CD
  - Containers
  - Amazon ECS
shortTitle: Deploy to Amazon ECS
ms.openlocfilehash: 259a3fd5bc0076f60d0c08f356b3ec9914effe89
ms.sourcegitcommit: fcf3546b7cc208155fb8acdf68b81be28afc3d2d
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 09/11/2022
ms.locfileid: '147881105'
---
{% data reusables.actions.enterprise-beta %} {% data reusables.actions.enterprise-github-hosted-runners %}

## Introduction

Ce guide explique comment utiliser {% data variables.product.prodname_actions %} pour créer une application conteneurisée, l’envoyer (push) à [Amazon Elastic Container Registry (INTUNE)](https://aws.amazon.com/ecr/) et la déployer sur [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/) en cas d’envoi (push) vers la branche `main`.

À chaque envoi (push) `main` dans votre référentiel {% data variables.product.company_short %}, le workflow {% data variables.product.prodname_actions %} génère et envoie (push) une nouvelle image conteneur vers Amazon ECR, puis déploie une nouvelle définition de tâche vers Amazon ECS.

{% ifversion fpt or ghec or ghae-issue-4856 or ghes > 3.4 %}

{% note %}

**Remarque** : {% data reusables.actions.about-oidc-short-overview %} et [« Configuration d’OpenID Connect dans Amazon Web Services »](/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services).

{% endnote %}

{% endif %}

## Prérequis

Avant de créer votre workflow {% data variables.product.prodname_actions %}, vous devez suivre les étapes de configuration suivantes pour Amazon ECR et ECS :

1. Créez un référentiel Amazon ECR pour stocker vos images.

   Par exemple à l’aide de l’interface [CLI AWS](https://aws.amazon.com/cli/) :

   {% raw %}```bash{:copy} aws ecr create-repository \
       --repository-name MY_ECR_REPOSITORY \
       --region MY_AWS_REGION
   ```{% endraw %}

   Ensure that you use the same Amazon ECR repository name (represented here by `MY_ECR_REPOSITORY`) for the `ECR_REPOSITORY` variable in the workflow below.

   Ensure that you use the same AWS region value for the `AWS_REGION` (represented here by `MY_AWS_REGION`) variable in the workflow below.

2. Create an Amazon ECS task definition, cluster, and service.

   For details, follow the [Getting started wizard on the Amazon ECS console](https://us-east-2.console.aws.amazon.com/ecs/home?region=us-east-2#/firstRun), or the [Getting started guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/getting-started-fargate.html) in the Amazon ECS documentation.

   Ensure that you note the names you set for the Amazon ECS service and cluster, and use them for the `ECS_SERVICE` and `ECS_CLUSTER` variables in the workflow below.

3. Store your Amazon ECS task definition as a JSON file in your {% data variables.product.company_short %} repository.

   The format of the file should be the same as the output generated by:

   {% raw %}```bash{:copy}
   aws ecs register-task-definition --generate-cli-skeleton
   ```{% endraw %}

   Ensure that you set the `ECS_TASK_DEFINITION` variable in the workflow below as the path to the JSON file.

   Ensure that you set the `CONTAINER_NAME` variable in the workflow below as the container name in the `containerDefinitions` section of the task definition.

4. Create {% data variables.product.prodname_actions %} secrets named `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to store the values for your Amazon IAM access key.

   For more information on creating secrets for {% data variables.product.prodname_actions %}, see "[Encrypted secrets](/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository)."

   See the documentation for each action used below for the recommended IAM policies for the IAM user, and methods for handling the access key credentials.

5. Optionally, configure a deployment environment. {% data reusables.actions.about-environments %}

## Creating the workflow

Once you've completed the prerequisites, you can proceed with creating the workflow.

The following example workflow demonstrates how to build a container image and push it to Amazon ECR. It then updates the task definition with the new image ID, and deploys the task definition to Amazon ECS.

Ensure that you provide your own values for all the variables in the `env` key of the workflow.

{% data reusables.actions.delete-env-key %}

```yaml{:copy}
{% data reusables.actions.actions-not-certified-by-github-comment %}

{% data reusables.actions.actions-use-sha-pinning-comment %}

name: Deploy to Amazon ECS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: MY_AWS_REGION                   # set this to your preferred AWS region, e.g. us-west-1
  ECR_REPOSITORY: MY_ECR_REPOSITORY           # set this to your Amazon ECR repository name
  ECS_SERVICE: MY_ECS_SERVICE                 # set this to your Amazon ECS service name
  ECS_CLUSTER: MY_ECS_CLUSTER                 # set this to your Amazon ECS cluster name
  ECS_TASK_DEFINITION: MY_ECS_TASK_DEFINITION # set this to the path to your Amazon ECS task definition
                                               # file, e.g. .aws/task-definition.json
  CONTAINER_NAME: MY_CONTAINER_NAME           # set this to the name of the container in the
                                               # containerDefinitions section of your task definition

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: {% data reusables.actions.action-checkout %}

      {% raw %}- name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@13d241b293754004c80624b5567555c4a39ffbe3
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@aaf69d68aa3fb14c1d5a6be9ac61fe15b48453a2

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build a docker container and
          # push it to ECR so that it can
          # be deployed to ECS.
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@97587c9d45a4930bf0e3da8dd2feb2a463cf4a3a
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@de0132cf8cdedb79975c6d42b77eb7ea193cf28e
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true{% endraw %}
```

## Ressources supplémentaires

Pour le workflow de démarrage d’origine, consultez [`aws.yml`](https://github.com/actions/starter-workflows/blob/main/deployments/aws.yml) dans le référentiel {% data variables.product.prodname_actions %} `starter-workflows`.

Pour plus d’informations sur les services utilisés dans ces exemples, consultez la documentation suivante :

* « [Meilleures pratiques de sécurité dans IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) » dans la documentation Amazon AWS.
* Action officielle AWS « [Configurer les informations d’identification AWS](https://github.com/aws-actions/configure-aws-credentials) ».
* Action officielle AWS [« Connexion » Amazon ECR](https://github.com/aws-actions/amazon-ecr-login).
* Action officielle AWS [« Définition de tâche de rendu » Amazon ECS.](https://github.com/aws-actions/amazon-ecs-render-task-definition)
* Action officielle AWS [« Définition de tâche de déploiement » Amazon ECS.](https://github.com/aws-actions/amazon-ecs-deploy-task-definition)
