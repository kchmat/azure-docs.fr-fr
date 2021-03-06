---
title: "Exécution et déclencheurs du pipeline dans Azure Data Factory | Microsoft Docs"
description: "Cet article fournit des informations sur l’exécution d’un pipeline dans Azure Data Factory, soit à la demande, soit en créant un déclencheur."
services: data-factory
documentationcenter: 
author: sharonlo101
manager: jhubbard
editor: spelluru
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 01/03/2018
ms.author: shlo
ms.openlocfilehash: fc34cfbab796c6e1e4cd25ce13dcc63c39c6699d
ms.sourcegitcommit: 79683e67911c3ab14bcae668f7551e57f3095425
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 01/25/2018
---
# <a name="pipeline-execution-and-triggers-in-azure-data-factory"></a>Exécution et déclencheurs du pipeline dans Azure Data Factory
> [!div class="op_single_selector" title1="Select the version of the Data Factory service that you're using:"]
> * [Version 1 - Disponibilité générale](v1/data-factory-scheduling-and-execution.md)
> * [Version 2 - Préversion](concepts-pipeline-execution-triggers.md)

Une _exécution du pipeline_ est un terme utilisé dans Azure Data Factory version 2 qui définit une instance d’une exécution du pipeline. Par exemple, supposons que vous disposez d’un pipeline qui s’exécute à 8h00, 9h00 et 10h00. Dans ce cas, il y aura trois exécutions du pipeline différentes. Chaque exécution du pipeline a une ID d’exécution de pipeline, correspondant à un GUID qui identifie de façon unique cette exécution de pipeline spécifique. Les exécutions de pipeline sont généralement instanciées en transmettant des arguments aux paramètres définis dans les pipelines. Il existe deux façons d’exécuter un pipeline : manuellement ou via un _déclencheur_. Cet article décrit de manière détaillée ces deux méthodes d’exécution d’un pipeline.

> [!NOTE]
> Cet article s’applique à Azure Data Factory version 2, actuellement en préversion. Si vous utilisez Azure Data Factory version 1, qui est en disponibilité générale (GA), consultez [Planification et exécution avec Azure Data Factory version 1](v1/data-factory-scheduling-and-execution.md).

## <a name="manual-execution-on-demand"></a>Exécution manuelle (à la demande)
L’exécution manuelle d’un pipeline est également appelée exécution _à la demande_.

Par exemple, supposons que vous souhaitez exécuter un pipeline simple appelé **copyPipeline**. Ce pipeline possède une seule activité qui copie à partir d’un dossier source situé dans le Stockage Blob Azure vers un dossier de destination situé au même endroit. La définition JSON suivante illustre cet exemple de pipeline :

```json
{
  "name": "copyPipeline",
  "properties": {
    "activities": [
      {
        "type": "Copy",
        "typeProperties": {
          "source": {
            "type": "BlobSource"
          },
          "sink": {
            "type": "BlobSink"
          }
        },
        "name": "CopyBlobtoBlob",
        "inputs": [
          {
            "referenceName": "sourceBlobDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "sinkBlobDataset",
            "type": "DatasetReference"
          }
        ]
      }
    ],
    "parameters": {
      "sourceBlobContainer": {
        "type": "String"
      },
      "sinkBlobContainer": {
        "type": "String"
      }
    }
  }
}
```

Selon la définition JSON, le pipeline accepte deux paramètres : **sourceBlobContainer** et **sinkBlobContainer**. Vous pouvez transmettre des valeurs à ces paramètres lors de l’exécution.

Vous pouvez exécuter manuellement votre pipeline à l’aide des méthodes suivantes :
- Le kit de développement logiciel (SDK) .NET.
- Le module Azure PowerShell.
- L’API REST.
- Le kit de développement logiciel (SDK) Python.

### <a name="the-rest-api"></a>API REST
L’exemple de commande suivant vous montre comment exécuter manuellement votre pipeline à l’aide de l’API REST :  

```
POST
https://management.azure.com/subscriptions/mySubId/resourceGroups/myResourceGroup/providers/Microsoft.DataFactory/factories/myDataFactory/pipelines/copyPipeline/createRun?api-version=2017-03-01-preview
```

Pour obtenir un exemple complet, consultez [Démarrage rapide : création d’une fabrique de données à l’aide de l’API REST](quickstart-create-data-factory-rest-api.md).

### <a name="azure-powershell"></a>Azure PowerShell
L’exemple de commande suivant vous montre comment exécuter manuellement votre pipeline à l’aide d’Azure PowerShell :

```powershell
Invoke-AzureRmDataFactoryV2Pipeline -DataFactory $df -PipelineName "Adfv2QuickStartPipeline" -ParameterFile .\PipelineParameters.json
```

Vous pouvez transmettre des paramètres dans le corps de la charge utile de la demande. Dans le kit de développement logiciel (SDK) .NET, Azure Powershell et le kit de développement logiciel (SDK) Python, les valeurs se transmettent dans un dictionnaire transmis sous forme d’argument à l’appel :

```json
{
  “sourceBlobContainer”: “MySourceFolder”,
  “sinkBlobCountainer”: “MySinkFolder”
}
```

La charge utile de réponse est un identificateur unique de l’exécution du pipeline :

```json
{
  "runId": "0448d45a-a0bd-23f3-90a5-bfeea9264aed"
}
```

Pour obtenir un exemple complet, consultez [Démarrage rapide : création d’une fabrique de données à l’aide d’Azure PowerShell](quickstart-create-data-factory-powershell.md).

### <a name="the-net-sdk"></a>Le kit de développement logiciel (SDK) .NET
L’exemple d’appel suivant vous montre comment exécuter manuellement votre pipeline à l’aide du kit de développement logiciel (SDK) .NET :

```csharp
client.Pipelines.CreateRunWithHttpMessagesAsync(resourceGroup, dataFactoryName, pipelineName, parameters)
```

Pour obtenir un exemple complet, consultez [Démarrage rapide : création d’une fabrique de données à l’aide du kit de développement logiciel (SDK) .NET](quickstart-create-data-factory-dot-net.md).

> [!NOTE]
> Vous pouvez utiliser le SDK .NET pour appeler des pipelines Azure Data Factory à partir d’Azure Functions, de vos services web, etc.

<h2 id="triggers">Exécution de déclencheur</h2>
Les déclencheurs permettent également de lancer une exécution du pipeline. Ils correspondent à une unité de traitement qui détermine le moment où une exécution de pipeline doit être lancée. Actuellement, Azure Data Factory prend en charge deux types de déclencheurs :
- Déclencheur de planification : un déclencheur qui appelle un pipeline selon un planning horaire.
- Déclencheur de fenêtre bascule : un déclencheur qui fonctionne sur un intervalle périodique, tout en conservant son état. Azure Data Factory ne prend actuellement pas en charge les déclencheurs d’événements. Par exemple, le déclencheur pour l’exécution d’un pipeline qui répond à un événement de réception d’un fichier.

Les pipelines et les déclencheurs ont une relation plusieurs-à-plusieurs. Plusieurs déclencheurs peuvent lancer un même pipeline et un seul déclencheur peut lancer plusieurs pipelines. Dans la définition de déclencheur suivante, la propriété **pipelines** fait référence à la liste des pipelines qui sont déclenchés par un déclencheur particulier. La définition de propriété inclut des valeurs pour les paramètres de pipeline.

### <a name="basic-trigger-definition"></a>Définition du déclencheur de base

```json
    "properties": {
        "name": "MyTrigger",
        "type": "<type of trigger>",
        "typeProperties": {
            …
        },
        "pipelines": [
            {
                "pipelineReference": {
                    "type": "PipelineReference",
                    "referenceName": "<Name of your pipeline>"
                },
                "parameters": {
                    "<parameter 1 Name>": {
                        "type": "Expression",
                          "value": "<parameter 1 Value>"
                    },
                    "<parameter 2 Name>" : "<parameter 2 Value>"
                }
            }
        ]
    }
```

## <a name="schedule-trigger"></a>Déclencheur de planification
Un déclencheur de planification exécute les pipelines selon un planning horaire. Ce déclencheur prend en charge les options de calendrier périodiques et avancées. Par exemple, le déclencheur prend en charge les intervalles comme « toutes les semaines » ou « lundi à 17h00 et jeudi à 21h00.» Le déclencheur de planification est flexible, car le modèle de jeu de données n’est pas spécifié et le déclencheur ne distingue pas les données de série chronologique des données de série non chronologique.

Pour plus d’informations sur les déclencheurs de planification et obtenir des exemples, consultez [Créer un déclencheur de planification](how-to-create-schedule-trigger.md).

## <a name="tumbling-window-trigger"></a>Déclencheur de fenêtre bascule
Les déclencheurs de fenêtre bascule sont un type de déclencheur qui s’active à un intervalle de temps périodique à partir d’une heure de début spécifiée, tout en conservant son état. Les fenêtres bascule sont une série d’intervalles de temps contigus fixes, qui ne se chevauchent pas. Pour plus d’informations sur les déclencheurs de fenêtre bascule et obtenir des exemples, consultez [Créer un déclencheur de fenêtre bascule](how-to-create-tumbling-window-trigger.md).

## <a name="schedule-trigger-definition"></a>Définition du déclencheur de planification
Quand vous créez un déclencheur de planification, vous spécifiez la planification et la périodicité à l’aide d’une définition JSON. 

Pour que votre déclencheur de planification lance une exécution de pipeline, insérez la référence du pipeline concerné dans la définition du déclencheur. Les pipelines et les déclencheurs ont une relation plusieurs-à-plusieurs. Plusieurs déclencheurs peuvent exécuter le même pipeline. Un seul déclencheur peut déclencher plusieurs pipelines.

```json
{
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": <<Minute, Hour, Day, Week, Year>>,
        "interval": <<int>>,             // How often to fire
        "startTime": <<datetime>>,
        "endTime": <<datetime>>,
        "timeZone": "UTC"
        "schedule": {                    // Optional (advanced scheduling specifics)
          "hours": [<<0-24>>],
          "weekDays": ": [<<Monday-Sunday>>],
          "minutes": [<<0-60>>],
          "monthDays": [<<1-31>>],
          "monthlyOccurences": [
               {
                    "day": <<Monday-Sunday>>,
                    "occurrence": <<1-5>>
               }
           ] 
        }
      }
    },
   "pipelines": [
            {
                "pipelineReference": {
                    "type": "PipelineReference",
                    "referenceName": "<Name of your pipeline>"
                },
                "parameters": {
                    "<parameter 1 Name>": {
                        "type": "Expression",
                        "value": "<parameter 1 Value>"
                    },
                    "<parameter 2 Name>" : "<parameter 2 Value>"
                }
           }
      ]
  }
}
```

> [!IMPORTANT]
> La propriété **parameters** est une propriété obligatoire de l’élément **pipelines**. Si votre pipeline n’accepte aucun paramètre, vous devez inclure une définition JSON vide pour la propriété **parameters**.

### <a name="schema-overview"></a>Vue d’ensemble du schéma
Le tableau suivant présente une vue d’ensemble globale des principaux éléments du schéma liés à la périodicité et à la planification d’un déclencheur :

| Propriété JSON | DESCRIPTION |
|:--- |:--- |
| **startTime** | Une valeur date-heure. Pour les planifications simples, la valeur de la propriété **startTime** s’applique à la première occurrence. Pour les planifications complexes, le déclencheur ne démarre pas avant la valeur **startTime** spécifiée. |
| **endTime** | La date et l’heure de fin du déclencheur. Le déclencheur ne s’exécute pas après la date et l’heure de fin spécifiées. La valeur de la propriété ne peut pas être dans le passé. <!-- This property is optional. --> |
| **timeZone** | Le fuseau horaire. Actuellement, seul le fuseau horaire UTC est pris en charge. |
| **recurrence** | Un objet de périodicité qui spécifie les règles de périodicité pour le déclencheur. L’objet de périodicité prend en charge les éléments suivants : **frequency**, **interval**, **endTime**, **count** et **schedule**. Lorsqu’un objet de périodicité est défini, l’élément **frequency** est requis. Les autres éléments de l’objet de périodicité sont facultatifs. |
| **frequency** | L’unité de fréquence à laquelle le déclencheur se répète. Les valeurs prises en charge incluent « minute », « heure », « day », « semaine » et « mois ». |
| **interval** | Un entier positif qui indique l’intervalle de la valeur **frequency**, qui détermine la fréquence d’exécution du déclencheur. Par exemple, si **l’intervalle** est défini sur 3 et la **fréquence** définie sur « semaine », le déclencheur se répète toutes les 3 semaines. |
| **schedule** | La planification périodique du déclencheur. Un déclencheur dont la valeur **frequency** spécifiée modifie sa périodicité selon une planification périodique. Une propriété **schedule** contient des modifications pour la périodicité basées sur des minutes, heures, jours de la semaine, jours du mois et numéro de semaine.

### <a name="schedule-trigger-example"></a>Exemple de déclencheur de planification

```json
{
    "properties": {
        "name": "MyTrigger",
        "type": "ScheduleTrigger",
        "typeProperties": {
            "recurrence": {
                "frequency": "Hour",
                "interval": 1,
                "startTime": "2017-11-01T09:00:00-08:00",
                "endTime": "2017-11-02T22:00:00-08:00"
            }
        },
        "pipelines": [{
                "pipelineReference": {
                    "type": "PipelineReference",
                    "referenceName": "SQLServerToBlobPipeline"
                },
                "parameters": {}
            },
            {
                "pipelineReference": {
                    "type": "PipelineReference",
                    "referenceName": "SQLServerToAzureSQLPipeline"
                },
                "parameters": {}
            }
        ]
    }
}
```

### <a name="schema-defaults-limits-and-examples"></a>Valeurs par défaut, limites et exemples du schéma

| Propriété JSON | type | Obligatoire | Valeur par défaut | Valeurs valides | exemples |
|:--- |:--- |:--- |:--- |:--- |:--- |
| **startTime** | Chaîne | OUI | Aucun | Dates-Heures ISO-8601 | `"startTime" : "2013-01-09T09:30:00-08:00"` |
| **recurrence** | Object | OUI | Aucun | Objet de périodicité | `"recurrence" : { "frequency" : "monthly", "interval" : 1 }` |
| **interval** | Number | Non  | 1 | 1 à 1 000 | `"interval":10` |
| **endTime** | Chaîne | OUI | Aucun | Une valeur date-heure représentant une heure dans le futur. | `"endTime" : "2013-02-09T09:30:00-08:00"` |
| **schedule** | Object | Non  | Aucun | Objet de planification | `"schedule" : { "minute" : [30], "hour" : [8,17] }` |

### <a name="starttime-property"></a>propriété startTime
Le tableau suivant vous montre comment la propriété **startTime** contrôle une exécution du déclencheur :

| Valeur startTime | Périodicité sans planification | Périodicité avec planification |
|:--- |:--- |:--- |
| Heure de début dans le passé | Calcule la prochaine exécution initiale après l’heure de début, puis l’exécute à l’heure indiquée.<br/><br/>Lance les exécutions suivantes à partir du calcul effectué lors de la dernière exécution.<br/><br/>Consultez l’exemple qui suit ce tableau. | Le déclencheur _ne démarre pas avant_ l’heure de début spécifiée. La première occurrence dépend de la planification calculée à partir de l’heure de début.<br/><br/>Lance les exécutions suivantes à partir de la planification de périodicité. |
| Heure de début dans le futur ou au moment présent | S’exécute une fois à l’heure de début spécifiée.<br/><br/>Lance les exécutions suivantes à partir du calcul effectué lors de la dernière exécution. | Le déclencheur _ne démarre pas avant_ l’heure de début spécifiée. La première occurrence dépend de la planification calculée à partir de l’heure de début.<br/><br/>Lance les exécutions suivantes à partir de la planification de périodicité. |

Voyons ce qui se passe lorsque l’heure de début se situe dans le passé, avec la valeur de périodicité définie mais pas la valeur de planification. Supposons que l’heure actuelle soit `2017-04-08 13:00`, l’heure de début `2017-04-07 14:00`, et que la périodicité soit tous les deux jours. (La valeur **recurrence** est définie en paramétrant la propriété **frequency** sur « jour » et la propriété **interval** sur 2.) Notez que la valeur **startTime** se situe dans le passé et se produit avant l’heure actuelle.

Dans ces conditions, la première exécution aura lieu le `2017-04-09 at 14:00`. Le moteur d'Azure Scheduler calcule les occurrences de l'exécution à partir de l'heure de début. Toutes les instances dans le passé sont ignorées. Le moteur utilise l'instance suivante qui se produit dans le futur. Dans ce scénario, l’heure de début étant `2017-04-07 at 2:00pm`, l’instance suivante aura lieu 2 jours après cette date, c’est-à-dire `2017-04-09 at 2:00pm`.

La première heure d’exécution est la même si la valeur **startTime** est `2017-04-05 14:00` ou `2017-04-01 14:00`. Après la première exécution, les exécutions suivantes sont calculées à l’aide de la planification. Par conséquent, les exécutions suivantes sont à `2017-04-11 at 2:00pm`, puis `2017-04-13 at 2:00pm`, puis `2017-04-15 at 2:00pm`, et ainsi de suite.

Enfin, lorsque les heures ou les minutes ne sont pas définies dans la planification d’un déclencheur, les heures ou minutes de la première exécution sont utilisées par défaut.

### <a name="schedule-property"></a>propriété de planification
D’une part, l’utilisation d’une planification peut limiter le nombre d’exécutions du déclencheur. Par exemple, si un déclencheur avec une fréquence mensuelle est planifié pour s’exécuter uniquement le 31, le déclencheur ne s’exécute que durant les mois qui comptent un 31e jour.

Une planification peut, quant à elle, également augmenter le nombre d’exécutions du déclencheur. Par exemple, un déclencheur avec une fréquence mensuelle planifiée pour s’exécuter le 1 et le 2 de chaque mois, s’exécute le 1 et le 2 de chaque mois, plutôt qu’une fois par mois.

Si plusieurs éléments **schedule** sont spécifiés, l’ordre d’évaluation va du plus grand au plus petit paramètre de planification. L’évaluation commence par le numéro de semaine, puis le jour du mois, jour de la semaine, heure et enfin, la minute.

Le tableau suivant décrit les éléments **schedule** en détail :

| Élément JSON | DESCRIPTION | Valeurs valides |
|:--- |:--- |:--- |
| **minutes** | Minutes d’exécution du déclencheur dans l’heure. | <ul><li>Entier </li><li>Tableau d’entiers</li></ul>
| **hours** | Heures d’exécution du déclencheur dans la journée. | <ul><li>Entier </li><li>Tableau d’entiers</li></ul> |
| **weekDays** | Jours d’exécution du déclencheur dans la semaine. La valeur ne peut être spécifiée qu’avec une fréquence hebdomadaire uniquement. | <ul><li>Lundi, mardi, mercredi, jeudi, vendredi, samedi, dimanche</li><li>Tableau des valeurs de jour (la taille maximale du tableau est de 7)</li><li>Les valeurs de jour ne respectent pas la casse</li></ul> |
| **monthlyOccurrences** | Jours d’exécution du déclencheur dans le mois. La valeur ne peut être spécifiée qu’avec une fréquence mensuelle uniquement. | <ul><li>Tableau d’objets **monthlyOccurence** : `{ "day": day,  "occurrence": occurence }`.</li><li>L’attribut **day** est le jour de la semaine durant lequel le déclencheur s’exécute. Par exemple, une propriété **monthlyOccurrences** avec une valeur **day** de `{Sunday}` signifie tous les dimanches du mois. L’attribut **day** est requis.</li><li>L’attribut **occurrence** est l’occurrence du **jour** spécifié au cours du mois. Par exemple, une propriété **monthlyOccurrences** avec les valeurs **day** et **occurrence** de `{Sunday, -1}` signifie le dernier dimanche du mois. L’attribut **occurrence** est facultatif.</li></ul> |
| **monthDays** | Jours d’exécution du déclencheur dans le mois. La valeur ne peut être spécifiée qu’avec une fréquence mensuelle uniquement. | <ul><li>Toute valeur <= -1 et >= -31</li><li>Toute valeur >= 1 et <= 31</li><li>Tableau de valeurs</li></ul> |

## <a name="examples-of-trigger-recurrence-schedules"></a>Exemples de planifications de périodicité de déclencheur
Vous trouverez dans cette section plusieurs exemples de planifications de périodicité qui se penchent tout particulièrement sur l’objet **schedule** et ses éléments.

Les exemples supposent que la valeur **interval** est 1 et que la valeur **frequency** est correcte selon la définition de planification. Par exemple, vous ne pouvez pas avoir une valeur **frequency** définie sur « jour » et une modification « monthDays » dans l’objet **schedule**. Ces restrictions sont mentionnées dans le tableau de la section précédente.

| exemples | DESCRIPTION |
|:--- |:--- |
| `{"hours":[5]}` | Exécution à 5h00 tous les jours. |
| `{"minutes":[15], "hours":[5]}` | Exécution à 5h15 tous les jours. |
| `{"minutes":[15], "hours":[5,17]}` | Exécution à 5h15 et 17h15 tous les jours. |
| `{"minutes":[15,45], "hours":[5,17]}` | Exécution à 5h15, 5h45, 17h15 et 17h45 tous les jours. |
| `{"minutes":[0,15,30,45]}` | Exécution toutes les 15 minutes. |
| `{hours":[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23]}` | Exécution toutes les heures. Ce déclencheur s’exécute toutes les heures. Les minutes sont contrôlées par la valeur **startTime**, lorsqu’une valeur est spécifiée. Si aucune valeur n’est spécifiée, les minutes sont contrôlées par l’heure de création. Par exemple, si l’heure de début ou l’heure de création (selon le cas) est 00h25, le déclencheur sera exécuté à 00h25, 01h25, 02h25, ..., et 23h25.<br/><br/>Cette planification est équivalente à un déclencheur avec une valeur **frequency** définie sur « heure», une valeur **interval** égale à 1 et aucune **planification**. Cette planification peut être utilisée avec des valeurs **frequency** et **interval** différentes pour créer d’autres déclencheurs. Par exemple, lorsque la valeur **frequency** est définie sur « mois », la planification ne s’exécute qu’une fois par mois, plutôt que tous les jours, lorsque la valeur **frequency** est définie sur « jour ». |
| `{"minutes":[0]}` | Exécution toutes les heures sur l’heure. Ce déclencheur s’exécute toutes les heures sur l’heure, en commençant à 00h00, 01h00, 02h00, et ainsi de suite.<br/><br/>Cette planification est équivalente à un déclencheur avec une valeur **frequency** définie sur « heure » et une valeur **startTime** de zéro minute, ou aucune **planification** mais une valeur **frequency** définie sur « jour ». Si la valeur **frequency** est définie sur « semaine » ou « mois », la planification s’exécute un jour par semaine ou un jour par mois seulement, respectivement. |
| `{"minutes":[15]}` | Exécution 15 minutes après l’heure, toutes les heures. Ce déclencheur s’exécute toutes les heures 15 minutes après l’heure, en commençant à 00h15, 01h15, 02h15, et ainsi de suite, jusqu’à 23h15. |
| `{"hours":[17], "weekDays":["saturday"]}` | Exécution à 17h le samedi chaque semaine. |
| `{"hours":[17], "weekDays":["monday", "wednesday", "friday"]}` | Exécution à 17h le lundi, le mercredi et le vendredi chaque semaine. |
| `{"minutes":[15,45], "hours":[17], "weekDays":["monday", "wednesday", "friday"]}` | Exécution à 17h15 et 17h45 le lundi, le mercredi et le vendredi chaque semaine. |
| `{"minutes":[0,15,30,45], "weekDays":["monday", "tuesday", "wednesday", "thursday", "friday"]}` | Exécution toutes les 15 minutes les jours de semaine. |
| `{"minutes":[0,15,30,45], "hours": [9, 10, 11, 12, 13, 14, 15, 16] "weekDays":["monday", "tuesday", "wednesday", "thursday", "friday"]}` | Exécution toutes les 15 minutes les jours de semaine entre 9h00 et 16h45. |
| `{"weekDays":["tuesday", "thursday"]}` | Exécution le mardi et le jeudi à l’heure de début spécifiée. |
| `{"minutes":[0], "hours":[6], "monthDays":[28]}` | Exécution à 6h00 le 28e jour de chaque mois (en supposant que la valeur **frequency** est définie sur « mois »). |
| `{"minutes":[0], "hours":[6], "monthDays":[-1]}` | Exécution à 6h00 le dernier jour du mois. Pour exécuter un déclencheur le dernier jour du mois, utilisez -1 au lieu des jours 28, 29, 30 ou 31. |
| `{"minutes":[0], "hours":[6], "monthDays":[1,-1]}` | Exécution à 6h00 le premier et le dernier jour de chaque mois. |
| `{monthDays":[1,14]}` | Exécution le premier et le 14e jour de chaque mois à l’heure de début spécifiée. |
| `{"minutes":[0], "hours":[5], "monthlyOccurrences":[{"day":"friday", "occurrence":1}]}` | Exécution le premier vendredi de chaque mois à 5h00. |
| `{"monthlyOccurrences":[{"day":"friday", "occurrence":1}]}` | Exécution le premier vendredi de chaque mois à l’heure de début spécifiée. |
| `{"monthlyOccurrences":[{"day":"friday", "occurrence":-3}]}` | Exécutez le troisième vendredi à partir de la fin du mois, chaque mois, à l’heure de début spécifiée. |
| `{"minutes":[15], "hours":[5], "monthlyOccurrences":[{"day":"friday", "occurrence":1},{"day":"friday", "occurrence":-1}]}` | Exécution le premier et le dernier vendredi de chaque mois à 5h15. |
| `{"monthlyOccurrences":[{"day":"friday", "occurrence":1},{"day":"friday", "occurrence":-1}]}` | Exécution le premier et le dernier vendredi de chaque mois à l’heure de début spécifiée. |
| `{"monthlyOccurrences":[{"day":"friday", "occurrence":5}]}` | Exécution le cinquième vendredi de chaque mois à l’heure de début spécifiée. S’il n’y a pas de cinquième vendredi dans un mois, le pipeline n’est pas exécuté, dans la mesure où il est planifié pour s’exécuter uniquement le cinquième vendredi du mois. Pour exécuter le déclencheur le dernier vendredi du mois, indiquez -1 au lieu de 5 pour la valeur **occurrence**. |
| `{"minutes":[0,15,30,45], "monthlyOccurrences":[{"day":"friday", "occurrence":-1}]}` | Exécution toutes les 15 minutes le dernier vendredi du mois. |
| `{"minutes":[15,45], "hours":[5,17], "monthlyOccurrences":[{"day":"wednesday", "occurrence":3}]}` | Exécution à 5h15, 5h45, 17h15 et 17h45 le troisième mercredi de chaque mois. |

## <a name="trigger-type-comparison"></a>Comparaison du type de déclencheur
Le déclencheur de fenêtre bascule et le déclencheur de planification fonctionnent tous les deux sur des pulsations de temps, donc en quoi sont-ils différents ?

Le tableau suivant présente une comparaison du déclencheur de fenêtre bascule et du déclencheur de planification :

|  | Déclencheur&nbsp;de fenêtre&nbsp;bascule | Déclencheur&nbsp;de planification |
|:--- |:--- |:--- |
| **Scénarios&nbsp;de renvoi** | Pris en charge. Les exécutions de pipeline peuvent être planifiées pour des fenêtres dans le passé. | Non pris en charge. Les exécutions de pipeline peuvent être exécutées uniquement sur des périodes de temps à partir de l’heure actuelle et dans le futur. |
| **Fiabilité** | Fiabilité de 100 %. Les exécutions de pipeline peuvent être planifiées pour toutes les fenêtres à partir d’une date de début spécifiée sans espaces. | Moins fiable. |
| **Capacité&nbsp;de nouvelle tentative** | Pris en charge. Les exécutions de pipeline ayant échoué ont une stratégie de nouvelle tentative par défaut d’une valeur de 0, ou une stratégie spécifiée par l’utilisateur dans le cadre de la définition du déclencheur. Réessaie automatiquement en cas d’échec des exécutions de pipeline en raison de limites de concurrence/serveur/limitation (autrement dit, les codes d’état 400 : Erreur de l’utilisateur, 429 : Trop de demandes et 500 : Erreur de serveur internet). | Non pris en charge. |
| **Concurrency** | Pris en charge. Les utilisateurs peuvent définir explicitement les limites de concurrence pour le déclencheur. Permet 1 à un maximum de 50 exécutions de pipeline déclenchées simultanées. | Non pris en charge. |
| **Variables&nbsp;système** | Prend en charge l’utilisation des variables système **WindowStart** et **WindowEnd**. Les utilisateurs peuvent accéder à `triggerOutputs().windowStartTime` et `triggerOutputs().windowEndTime` comme variables système de déclencheur dans la définition du déclencheur. Les valeurs sont utilisées en tant qu’heure de début de fenêtre et heure de fin de fenêtre, respectivement. Par exemple, pour un déclencheur de fenêtre bascule qui s’exécute toutes les heures, pour la fenêtre de 1h00 à 2h00, la définition est `triggerOutputs().WindowStartTime = 2017-09-01T01:00:00Z` et `triggerOutputs().WindowEndTime = 2017-09-01T02:00:00Z`. | Non pris en charge. |
| **Relation&#x2011;du&#x2011;pipeline et du déclencheur** | Prend en charge une relation un à un. Un seul pipeline peut être déclenché. | Prend en charge les relations plusieurs à plusieurs. Plusieurs déclencheurs peuvent exécuter le même pipeline. Un seul déclencheur peut déclencher plusieurs pipelines. | 

## <a name="next-steps"></a>étapes suivantes
Consultez les didacticiels suivants :

- [Démarrage rapide : Créer une fabrique de données à l’aide du SDK .NET](quickstart-create-data-factory-dot-net.md)
- [Créer un déclencheur de planification](how-to-create-schedule-trigger.md)
- [Créer un déclencheur de fenêtre bascule](how-to-create-tumbling-window-trigger.md)
