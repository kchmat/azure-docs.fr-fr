---
title: "Apache Spark Structured Streaming de Kafka à Azure Cosmos DB - Azure HDInsight | Microsoft Docs"
description: "Découvrez comment utiliser Apache Spark Structured Streaming pour lire des données à partir d’Apache Kafka puis les stocker dans Azure Cosmos DB. Dans cet exemple, vous diffusez des données à l’aide d’un bloc-notes Jupyter à partir de Spark sur HDInsight."
services: hdinsight
documentationcenter: 
author: Blackmist
manager: jhubbard
editor: cgronlun
ms.service: hdinsight
ms.custom: hdinsightactive
ms.devlang: 
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: big-data
ms.date: 01/16/2018
ms.author: larryfr
ms.openlocfilehash: 55d0fb91c8a8b995a5b9369d762f5bd87cb086c9
ms.sourcegitcommit: 9890483687a2b28860ec179f5fd0a292cdf11d22
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 01/24/2018
---
# <a name="use-spark-structured-streaming-with-kafka-and-azure-cosmos-db"></a>Utiliser Spark Structured Streaming avec Kafka et Azure Cosmos DB

Découvrez comment utiliser Spark Structured Streaming pour lire des données à partir d’Apache Kafka sur Azure HDInsight, puis les stocker dans Azure Cosmos DB.

Azure Cosmos DB est une base de données multimodèle distribuée à l’échelle mondiale. Cet exemple utilise un modèle de base de données d’API SQL. Pour plus d’informations, consultez le document [Bienvenue dans Azure Cosmos DB](../cosmos-db/introduction.md).

Spark Structured Streaming est un moteur de traitement de flux basé sur Spark SQL. Il vous permet d’exprimer des calculs de diffusion en continu de la même façon que pour les calculs de lot sur les données statiques. Pour plus d’informations sur Structured Streaming, consultez le [Guide de programmation Structured Streaming [Alpha]](http://spark.apache.org/docs/2.1.0/structured-streaming-programming-guide.html) sur le site Apache.org.

> [!IMPORTANT]
> Cet exemple utilise Spark 2.1 sur HDInsight 3.6. Structured Streaming est considéré comme __alpha__ sur Spark 2.1.
>
> Les étapes décrites dans ce document créent un groupe de ressources Azure qui contient à la fois un Spark sur HDInsight et un Kafka sur un cluster HDInsight. Ces clusters sont tous deux situés dans un réseau virtuel Azure, ce qui permet au cluster Spark de communiquer directement avec le cluster Kafka.
>
> Lorsque vous avez terminé les étapes décrites dans ce document, n’oubliez pas de supprimer les clusters pour éviter les frais supplémentaires.

## <a name="create-the-clusters"></a>Création des clusters

Apache Kafka sur HDInsight ne donne pas accès aux répartiteurs Kafka via l’internet public. Tout ce qui communique avec Kafka doit se trouver sur le même réseau virtuel Azure que les nœuds du cluster Kafka. Pour cet exemple, les clusters Kafka et Spark sont situés dans un réseau virtuel Azure. Le diagramme suivant illustre le flux de communication entre les clusters :

![Diagramme des clusters Spark et Kafka dans un réseau virtuel Azure](./media/hdinsight-apache-spark-with-kafka/spark-kafka-vnet.png)

> [!NOTE]
> Le service Kafka est limité à la communication au sein du réseau virtuel. L’accès aux autres services sur le cluster, tels que SSH et Ambari, se fait via Internet. Pour plus d’informations sur les ports publics disponibles avec HDInsight, consultez [Ports et URI utilisés par HDInsight](hdinsight-hadoop-port-settings-for-services.md).

Même si vous pouvez créer un réseau virtuel Azure, et des clusters Kafka et Spark manuellement, il est plus facile d’utiliser un modèle Azure Resource Manager. Utilisez les étapes suivantes pour déployer un réseau virtuel Azure et des clusters Kafka et Spark sur votre abonnement Azure.

1. Utilisez le bouton suivant pour vous connecter à Azure et ouvrir le modèle dans le portail Azure.
    
    <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fhdinsight-spark-scala-kafka-cosmosdb%2Fmaster%2Fazuredeploy.json" target="_blank"> <img src="http://azuredeploy.net/deploybutton.png"/> </a>

    Le modèle Azure Resource Manager se trouve dans le référentiel GitHub pour ce projet ([https://github.com/Azure-Samples/hdinsight-spark-scala-kafka-cosmosdb](https://github.com/Azure-Samples/hdinsight-spark-scala-kafka-cosmosdb)).

    Ce modèle crée les ressources suivantes :

    * Un cluster Kafka sur HDInsight 3.6.

    * Un cluster Spark sur HDInsight 3.6.

    * Un réseau virtuel Azure, qui contient les clusters HDInsight.

    * Une base de données d’API SQL Azure Cosmos DB.

    > [!IMPORTANT]
    > Le bloc-notes de diffusion en continu structurée utilisé dans cet exemple nécessite Spark sur HDInsight 3.6. Si vous utilisez une version antérieure de Spark sur HDInsight, vous recevez des erreurs lors de l’utilisation du bloc-notes.

2. Utilisez les informations suivantes pour remplir les entrées dans la section **Déploiement personnalisé** :
   
    ![Déploiement HDInsight personnalisé](./media/hdinsight-apache-spark-with-kafka/parameters.png)
   
    * **Groupe de ressources** : créez un groupe de ressources ou sélectionnez-en un. Ce groupe contient le cluster HDInsight.

    * **Emplacement** : choisissez un emplacement proche de vous géographiquement.

    * **Nom de cluster de base** : cette valeur sera utilisée comme nom de base pour les clusters Spark et Kafka. Par exemple, si vous entrez **hdi**, cela crée un cluster Spark nommé spark-hdi__ et un cluster Kafka nommé **kafka-hdi**.

    * **Nom d’utilisateur du cluster** : nom utilisateur Admin pour les clusters Spark et Kafka.

    * **Mot de passe du cluster** : mot de passe utilisateur Admin pour les clusters Spark et Kafka.

    * **Nom d’utilisateur SSH** : utilisateur SSH permettant de créer des clusters Spark et Kafka.

    * **Mot de passe SSH** : mot de passe de l’utilisateur SSH pour les clusters Spark et Kafka.

3. Passez en revue les **termes et conditions**, puis cochez la case **J’accepte les termes et conditions mentionnés ci-dessus**.

4. Pour finir, cochez **Épingler au tableau de bord**, puis sélectionnez **Acheter**. La création des clusters prend environ 20 minutes.

Dès que les ressources sont créées, une page récapitulative s’affiche.

![Informations sur le groupe de ressources pour le réseau virtuel et les clusters](./media/hdinsight-apache-spark-with-kafka/groupblade.png)

> [!IMPORTANT]
> Les noms des clusters HDInsight sont **spark-BASENAME** et **kafka-BASENAME**, où BASENAME est le nom que vous avez fourni pour le modèle. Vous utilisez ces noms dans les étapes ultérieures, lors de la connexion aux clusters.

## <a name="get-the-kafka-brokers"></a>Obtenir les répartiteurs Kafka

Le code dans cet exemple se connecte aux hôtes répartiteurs Kafka dans le cluster Kafka. Pour rechercher les adresses des deux hôtes répartiteurs Kafka, utilisez l’exemple PowerShell ou Bash suivant :

```powershell
$creds = Get-Credential -UserName "admin" -Message "Enter the HDInsight login"
$clusterName = Read-Host -Prompt "Enter the Kafka cluster name"
$resp = Invoke-WebRequest -Uri "https://$clusterName.azurehdinsight.net/api/v1/clusters/$clusterName/services/KAFKA/components/KAFKA_BROKER" `
    -Credential $creds
$respObj = ConvertFrom-Json $resp.Content
$brokerHosts = $respObj.host_components.HostRoles.host_name[0..1]
($brokerHosts -join ":9092,") + ":9092"
```

```bash
curl -u admin -G "https://$CLUSTERNAME.azurehdinsight.net/api/v1/clusters/$CLUSTERNAME/services/KAFKA/components/KAFKA_BROKER" | jq -r '["\(.host_components[].HostRoles.host_name):9092"] | join(",")' | cut -d',' -f1,2
```

Lorsque vous y êtes invité, entrez le mot de passe du compte de connexion (admin) au cluster.

> [!NOTE]
> Cet exemple attend `$CLUSTERNAME` pour contenir le nom du cluster Kafka.
>
> Cet exemple utilise l’utilitaire [jq](https://stedolan.github.io/jq/) pour analyser les données du document JSON.

Le résultat ressemble au texte suivant :

`wn0-kafka.0owcbllr5hze3hxdja3mqlrhhe.ex.internal.cloudapp.net:9092,wn1-kafka.0owcbllr5hze3hxdja3mqlrhhe.ex.internal.cloudapp.net:9092`

Notez ces informations car elles sont utilisées dans les sections suivantes de ce document.

## <a name="get-the-notebooks"></a>Obtenir les blocs-notes

Le code de l’exemple décrit dans ce document est disponible à l’adresse [https://github.com/Azure-Samples/hdinsight-spark-scala-kafka-cosmosdb](https://github.com/Azure-Samples/hdinsight-spark-scala-kafka-cosmosdb).

## <a name="upload-the-notebooks"></a>Charger les blocs-notes

Pour charger les blocs-notes à partir du projet vers votre cluster Spark sur HDInsight, procédez comme suit :

1. Dans votre navigateur web, connectez-vous au bloc-notes Jupyter sur votre cluster Spark. Dans l’URL suivante, remplacez `CLUSTERNAME` par le nom de votre cluster __Spark__ :

        https://CLUSTERNAME.azurehdinsight.net/jupyter

    Lorsque vous y êtes invité, entrez l’identifiant de connexion (admin) et le mot de passe du cluster utilisés lors de la création du cluster.

2. Dans la partie supérieure droite de la page, utilisez le bouton __Charger__ pour charger le fichier __Stream-taxi-data-to-kafka.ipynb__ dans le cluster. Sélectionnez __Ouvrir__ pour démarrer le chargement.

3. Recherchez l’entrée __Stream-taxi-data-to-kafka.ipynb__ dans la liste des blocs-notes, puis sélectionnez le bouton __Charger__ en regard de celle-ci.

4. Répétez les étapes 1 à 3 pour charger le bloc-notes __Stream-data-from-Kafka-to-Cosmos-DB.ipynb__.

## <a name="load-taxi-data-into-kafka"></a>Charger des données de taxi dans Kafka

Une fois que les fichiers ont été chargés, sélectionnez l’entrée __Stream-taxi-data-to-kafka.ipynb__ pour ouvrir le bloc-notes. Suivez les étapes dans le bloc-notes pour charger des données dans Kafka.

## <a name="process-taxi-data-using-spark-structured-streaming"></a>Traiter des données de taxi à l’aide de Spark Structured Streaming

Dans la page d’accueil du bloc-notes Jupyter, sélectionnez l’entrée __Stream-data-from-Kafka-to-Cosmos-DB.ipynb__. Suivez les étapes dans le bloc-notes pour transmettre en continu des données à partir de Kafka dans Azure Cosmos DB à l’aide de Spark Structured Streaming.

## <a name="next-steps"></a>étapes suivantes

Maintenant que vous avez appris à utiliser Spark Structured Streaming, consultez les documents suivants pour en savoir plus sur l’utilisation de Spark, Kafka et Azure Cosmos DB :

* [Utilisation de la diffusion en continu Spark (DStream) avec Kafka](hdinsight-apache-spark-with-kafka.md).
* [Prise en main du bloc-notes Jupyter et Spark sur HDInsight](spark/apache-spark-jupyter-spark-sql.md)
* [Bienvenue dans Azure Cosmos DB](../cosmos-db/introduction.md)
