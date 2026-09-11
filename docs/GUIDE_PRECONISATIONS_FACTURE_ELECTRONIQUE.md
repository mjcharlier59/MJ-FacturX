# Facturation électronique française — Pièges d'intégration

*Chorus Pro / PDP-Peppol : blocages rencontrés en conditions réelles*

Ce document ne montre aucun code — il résume des blocages concrets rencontrés lors de l'intégration d'un logiciel tiers avec Chorus Pro (facturation B2G, secteur public) et une Plateforme Agréée (PDP) compatible Peppol (facturation B2B), dans le cadre de la réforme française de facturation électronique. L'objectif est de faire gagner du temps à d'autres développeurs qui buteraient sur les mêmes points — la documentation officielle est correcte sur le fond, mais souvent en décalage avec l'interface actuelle ou incomplète sur des cas limites.

---

## 📋 Contexte

Deux flux distincts à gérer, avec des logiques et des identifiants différents :

- **B2G** (vers le secteur public) → Chorus Pro, via l'API PISTE (OAuth2 client credentials + un « compte technique » applicatif).
- **B2B** (entre entreprises) → une Plateforme Agréée (PDP) qui relaie les factures sur le réseau Peppol, avec un adressage basé sur des identifiants normalisés (SIREN/SIRET), pas sur une simple adresse e-mail.

Les deux réglementations avancent en parallèle mais les mécanismes techniques ne se ressemblent pas — attention à ne pas transposer une logique de l'un vers l'autre.

---

## 🔴 Chorus Pro : le portail a changé, pas toujours la doc

Les guides AIFE historiques (2017-2019) parlent d'un onglet « Activités du gestionnaire » pour tout ce qui touche à la gestion de structure. Sur le portail actuel (post-refonte « portail de services »), ces fonctions sont réparties dans des domaines distincts, et le nom « Activités du gestionnaire » ne s'y retrouve pas forcément tel quel.

### Création d'un compte technique

**Chemin :** Domaine Raccordements → application Compte technique → type de demande « Création d'un compte technique » → choisir la structure.

⚠️ **Piège :** Le compte n'est actif qu'après ~30 minutes — un test immédiat après création peut donc échouer alors que la procédure s'est bien déroulée.

### Déclaration / Vérification d'une coordonnée bancaire (RIB)

**Chemin :** Domaine Organisation → application Structures → « Créer une coordonnée bancaire ».

⚠️ **Piège :** Sans RIB valide enregistré, certains appels API renvoient une réponse HTTP 200 mais un `codeRetour` métier signalant qu'aucun résultat n'a été trouvé — ce n'est pas un bug côté intégration, c'est un vrai prérequis manquant côté configuration Chorus Pro.

### Raccordement API et compte technique

Le **raccordement API** (association d'une application PISTE à une structure) est une étape distincte de la création du compte technique — **les deux sont nécessaires**, ni l'un ni l'autre ne remplace l'autre.

### Piège transverse : mélange sandbox/production

Si votre outil gère plusieurs structures/clients avec des jeux d'identifiants différents (sandbox vs production, client A vs client B), assurez-vous que tout ce qui identifie une structure (SIRET, compte technique, éventuel identifiant interne) provient bien du **même environnement/client** au moment du test.

Un mélange entre un SIRET resté sur une valeur de test et des identifiants applicatifs de production produit des erreurs qui ressemblent à un problème d'habilitation API, alors que la cause réelle est un simple décalage de configuration.

**Symptôme le plus trompeur :** Une erreur HTTP 403 « sèche », sans enveloppe de réponse métier — c'est en général le signe d'un rejet au niveau de la passerelle avant même d'atteindre la validation applicative, cohérent avec un jeton et une donnée métier qui ne correspondent pas à la même structure.

---

## 🌐 SuperPDP / Peppol : l'adresse n'est pas juste un SIREN

Pour une PDP compatible Peppol, chaque acteur (émetteur et destinataire) est identifié par un **participant ID** au format `schéma:valeur` (norme ISO 6523).

Pour la France, le schéma dédié à la réforme de facturation électronique est **0225** (à ne pas confondre avec le schéma 0002 utilisé, lui, pour l'identification légale de l'entité dans le corps UBL — **les deux coexistent et ont des rôles différents**).

### Erreur fréquente : supposer que l'adresse Peppol = 0225:SIREN

En pratique :

- ✅ C'est vrai pour une entreprise mono-établissement enregistrée telle quelle.
- ❌ Mais la spécification prévoit aussi des compositions `SIREN_SIRET` (pour cibler un établissement précis) ou `SIREN_SIRET_<code routage>` (pour un service particulier) — **deviner l'adresse à partir du seul SIREN peut donc échouer** ou, pire, router vers la mauvaise entité pour un client multi-sites.
- ❌ Certaines entreprises apparaissent dans l'annuaire avec **plusieurs adresses**, dont des variantes techniques auxiliaires (repérables par un suffixe explicite du type `_replyto`) qui servent uniquement aux accusés de réception, pas à la réception de factures.

### Solution robuste : interroger l'annuaire Peppol

Un moyen fiable d'écarter les mauvaises adresses : vérifier les `docTypes` associés à chaque entrée — seule une adresse listant un type `Invoice/CreditNote` conforme au profil de facturation doit être utilisée pour l'envoi.

**Interroger l'annuaire Peppol officiel avant l'envoi** plutôt que de construire l'adresse en dur. L'annuaire expose une API REST publique, sans authentification :

```
GET https://directory.peppol.eu/search/1.0/json?q=<SIREN ou nom ou n° TVA>
```

La réponse liste les correspondances avec, pour chacune, un `participantID` (scheme + value) et un tableau `docTypes`.

**Avantages :**
- Confirme qu'un client donné est bien joignable sur le réseau Peppol avant de tenter un envoi
- Évite de découvrir l'échec après coup
- Un outil web gratuit ([peppolcheck.fr](https://peppolcheck.fr)) permet de faire la même recherche manuellement pour du debug ponctuel

### Sandbox vs Production sur une même URL

Certaines PDP (c'est le cas de SUPER PDP) utilisent la **même URL d'API pour le bac-à-sable et la production** — la bascule se fait uniquement via les **identifiants OAuth** de l'application (chaque environnement correspond à une application distincte chez le PDP), pas via un sous-domaine ou un chemin différent.

Vérifiez ce point spécifiquement pour votre PDP avant de supposer une convention `sandbox-api.*` comme c'est le cas chez d'autres fournisseurs (dont Chorus Pro/PISTE, qui eux distinguent bien deux domaines).

---

## 📌 Recommandations générales

### 1. Journaliser les réponses brutes, pas juste le code HTTP

Tester un appel API qui échoue en HTTP 200 avec un `codeRetour` métier différent de 0 n'est pas la même chose qu'un échec HTTP — dans les deux cas, **journaliser la réponse brute complète** (pas seulement le code HTTP) fait gagner un temps considérable en debug, surtout quand la cause est une donnée de configuration plutôt qu'un bug de code.

### 2. Vérifier contre un registre officiel

De façon générale, avant de coder une hypothèse de format d'identifiant (adresse Peppol, structure, etc.), il vaut mieux la **vérifier contre un registre officiel en conditions réelles** que de se fier à un exemple trouvé dans un guide, aussi récent soit-il — ces plateformes évoluent vite en ce moment, la réforme étant encore en déploiement.

---

## 🔗 Liens utiles

- [Documentation Chorus Pro (portail officiel)](https://www.chorus-pro.gouv.fr)
- [Annuaire Peppol officiel — recherche](https://directory.peppol.eu/search/1.0/json)
- [PeppolCheck.fr — recherche gratuite dans l'annuaire Peppol français](https://peppolcheck.fr)
- [Documentation SUPER PDP](https://www.super-pdp.fr)

---

**Dernière mise à jour :** 2026-09-11  
**Auteur :** MJ-FacturX
