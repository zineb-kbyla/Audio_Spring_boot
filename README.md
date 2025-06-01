
# Système de Text-to-Speech (TTS) en Temps Réel – Bewize

## Introduction
Ce document présente un résumé technique et fonctionnel du système de synthèse vocale (Text-to-Speech, TTS) intégré à l’application éducative **Bewize**. Ce système permet de transformer dynamiquement des contenus textuels (flashcards, quizzcards) en audio afin de favoriser l’inclusion, l’accessibilité et l’interactivité.

## Architecture du Système
Le système TTS s’appuie sur une architecture distribuée et modulaire :

- **Frontend (React)** : Interface utilisateur riche, interactive, où l’utilisateur peut écouter la lecture vocale des **flashcards** (questions/réponses) et des **quizzcards** (questions/feedback) directement depuis la base de données.
- **Backend (Spring Boot)** : Serveur d’application centralisé qui gère la logique métier, construit les requêtes vers le service vocal et orchestre les retours.
- **API ElevenLabs** : Fournisseur de services TTS dans le cloud, capable de générer de la voix à partir de texte multilingue avec un rendu naturel.
- **Base de Données** : Utilisée pour stocker les éléments pédagogiques textuels et, éventuellement, les fichiers audios générés à des fins de réutilisation ou d’analyse.

## Fonctionnement Général
Le processus temps réel se déroule comme suit :

1. L’utilisateur déclenche une lecture vocale (via un bouton) sur une **flashcard** ou une **quizzcard**.
2. Le frontend envoie une requête contenant le **texte à lire**, la **langue souhaitée**, et des **paramètres personnalisés** (ex. : type de voix) au backend.
3. Le backend prépare et transmet la requête vers l’API ElevenLabs.
4. L’API génère l’audio correspondant et retourne un fichier de type `MP3`.
5. Le backend renvoie ce fichier vers le frontend, qui l’exécute instantanément dans le navigateur de l’utilisateur.

## Technologies Utilisées
- **ReactJS** : Composants dynamiques et interactifs pour la lecture audio.
- **Spring Boot (Java)** : Traitement sécurisé des requêtes, gestion des API externes.
- **Howler.js** : Librairie JavaScript pour la lecture efficace et contrôlée des fichiers audio.
- **ElevenLabs API** : Plateforme de synthèse vocale en ligne avec support multilingue, voix expressives et personnalisables.

## Sécurité et Optimisations
Le système TTS intègre plusieurs mesures de sécurité et d’optimisation :

### Sécurité
- Validation et nettoyage des textes envoyés pour éviter les injections ou abus.
- Limitation de la taille des messages (ex. : 5000 caractères maximum).
- Utilisation exclusive de connexions **HTTPS** pour chiffrer les communications.
- Protection contre l’abus avec une limitation du nombre de requêtes (rate limiting).

### Optimisation
- **Mise en cache** côté serveur pour éviter les appels répétés sur les textes déjà générés.
- Pré-chargement et pré-génération des audios pour les contenus statiques ou les sessions intensives.
- Affichage d’indicateurs de chargement pour offrir une meilleure expérience utilisateur.

## Dépannage et Gestion des Erreurs
Voici quelques pistes de diagnostic :

- **Erreur 401 (Unauthorized)** : Vérifier la validité de la clé API ElevenLabs.
- **Temps de réponse lent** : Réduire la taille du texte ou utiliser un système de pré-génération.
- **Absence de son** : Confirmer que le format audio est pris en charge (MP3) par le navigateur.
- **Erreur côté serveur** : Analyser les logs pour identifier les exceptions au niveau des appels API.

## Structure Backend (Spring Boot)

Voici un aperçu organisationnel de la structure du projet côté backend :

```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── bewize/
│   │           ├── controller/
│   │           │   └── TTSController.java        # Contrôleur REST exposant l’endpoint TTS
│   │           ├── service/
│   │           │   └── TTSService.java           # Service dédié à l’appel ElevenLabs
│   │           └── dto/
│   │               └── TTSRequest.java           # Objet de transfert contenant texte + langue
│   └── resources/
│       ├── application.properties                # Contient la clé API et l’URL de ElevenLabs
│       └── static/                               # (optionnel) fichiers statiques
└── test/
    └── java/
        └── com.bewize/                           # Tests unitaires et d’intégration
```

## Intégration dans Spring Boot

Le composant TTS est intégré comme suit dans l’architecture Spring Boot :

- **TTSController** : Expose un point d’entrée `/api/tts/generate` en POST.
- **TTSService** (recommandé) : Contient la logique d’appel vers l’API ElevenLabs. Ce découpage permet une séparation des responsabilités (Controller vs logique métier).
- **TTSRequest** : Classe simple utilisée pour transférer le contenu textuel et la langue depuis le frontend.
- **application.properties** : Fichier de configuration dans lequel la clé API et l’URL du service TTS sont stockées, typiquement :
  ```
  elevenlabs.api.key=your-api-key
  elevenlabs.api.url=https://api.elevenlabs.io/v1/text-to-speech
  ```

Cette architecture permet de maintenir une application modulaire, testable, et facile à faire évoluer.

## Perspectives d’Amélioration
Le système a été conçu pour évoluer facilement :

- Permettre à l’utilisateur de **choisir sa voix préférée** parmi un catalogue de voix multilingues.
- Ajouter des **paramètres ajustables** : vitesse de lecture, intonation, expressivité.
- Mettre en place un **historique d’audios générés** pour une réécoute ou un suivi pédagogique.
- Étudier l’intégration d’un moteur **TTS local (offline)** pour les environnements sans connexion Internet.

---

**Conclusion** : Le système TTS pour Bewize représente une réelle avancée technologique et pédagogique, en rendant les contenus accessibles à tous, particulièrement aux personnes en situation de handicap visuel ou ayant des difficultés de lecture. Il s’inscrit pleinement dans une démarche inclusive et innovante.
