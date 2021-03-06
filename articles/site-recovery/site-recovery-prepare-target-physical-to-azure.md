---
title: "Préparer la cible (Physique vers Azure) | Microsoft Docs"
description: "Cet article décrit comment préparer votre environnement Azure pour lancer la réplication des serveurs physiques exécutant Windows ou Linux vers Azure."
services: site-recovery
documentationcenter: 
author: bsiva
manager: abhemraj
editor: 
ms.assetid: 
ms.service: site-recovery
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: backup-recovery
ms.date: 11/23/2017
ms.author: bsiva
ms.openlocfilehash: 2c5377f7193f8357a7e99ed1ef1a61b066b8ce5f
ms.sourcegitcommit: b5c6197f997aa6858f420302d375896360dd7ceb
ms.translationtype: HT
ms.contentlocale: fr-FR
ms.lasthandoff: 12/21/2017
---
# <a name="prepare-target-vmware-to-azure"></a>Préparer la cible (VMware vers Azure)
> [!div class="op_single_selector"]
> * [VMware vers Azure](./site-recovery-prepare-target-vmware-to-azure.md)
> * [Physique vers Azure](./site-recovery-prepare-target-physical-to-azure.md)

Cet article décrit comment préparer votre environnement Azure pour lancer la réplication des serveurs physiques (x64) exécutant Windows ou Linux vers Azure.

## <a name="prerequisites"></a>Composants requis

L’article suppose que :
- Vous avez créé un coffre Recovery Services pour protéger vos serveurs physiques. Vous pouvez créer un coffre Recovery Services dans le [portail Azure](http://portal.azure.com "portail Azure").
- Vous avez [configuré votre environnement local](./site-recovery-set-up-physical-to-azure.md) pour répliquer les serveurs physiques vers Azure.

## <a name="prepare-target"></a>Préparer la cible

Après avoir réalisé **l’Étape 1 : Sélection de l’objectif de la protection** et **l’Étape 2 : Préparation de la source**, vous passez à **l’Étape 3 : Cible**.

![Préparer la cible](./media/site-recovery-prepare-target-physical-to-azure/prepare-target-physical-to-azure.png)

1. **Abonnement :** dans le menu déroulant, sélectionnez l’abonnement vers lequel vous souhaitez répliquer vos serveurs physiques.
2. **Modèle de déploiement :** sélectionnez le modèle de déploiement (Classique ou Gestionnaire de ressources).

Selon le modèle de déploiement choisi, une validation est exécutée pour vérifier que vous disposez au moins d’un compte de stockage et d’un réseau virtuel compatibles dans l’abonnement cible pour la réplication et le basculement de vos serveurs physiques.

Une fois les validations terminées avec succès, cliquez sur OK pour passer à l’étape suivante.

Si vous n’avez pas de compte de stockage Resource Manager ou de réseau virtuel compatible, vous pouvez en créer un en cliquant sur le bouton **+ Compte de stockage** ou sur le bouton **+ Réseau** en haut de la page.

## <a name="next-steps"></a>Étapes suivantes
[Configurer les paramètres de réplication](./site-recovery-setup-replication-settings-vmware.md)
