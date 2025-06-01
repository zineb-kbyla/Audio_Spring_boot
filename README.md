
# Système de Text-to-Speech (TTS) en temps réel pour Bewize

## Table des Matières
- Introduction
- Architecture du Système
- Intégration Frontend (React)
- Intégration Backend (Spring Boot)
- Configuration ElevenLabs
- Flux de Travail Temps Réel
- Sécurité et Optimisation
- Dépannage
- Évolution Future

## Introduction
Ce document explique comment implémenter un système de Text-to-Speech (TTS) en temps réel utilisant l'API ElevenLabs dans l'application Bewize, à la fois pour le frontend React et le backend Spring Boot.

## Architecture du Système
**Diagramme d'architecture**
- **Frontend (React)** : Gère l'interface utilisateur et les interactions
- **Backend (Spring Boot)** : Serveur d'application qui communique avec ElevenLabs
- **API ElevenLabs** : Service de synthèse vocale
- **Base de données** : Stocke les références des fichiers audio générés

## Intégration Frontend (React)
### Installation des Dépendances
```bash
npm install axios howler react-icons
```

### Composant AudioPlayer
```jsx
import React, { useState, useRef } from 'react';
import { FaPlay, FaStop } from 'react-icons/fa';
import axios from 'axios';
import { Howl } from 'howler';

const AudioPlayer = ({ text, language }) => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const soundRef = useRef(null);

  const handlePlay = async () => {
    if (isPlaying) {
      soundRef.current?.stop();
      setIsPlaying(false);
      return;
    }

    setIsLoading(true);
    try {
      const response = await axios.post('/api/tts/generate', {
        text,
        language
      }, {
        responseType: 'blob'
      });

      const audioBlob = new Blob([response.data], { type: 'audio/mpeg' });
      const audioUrl = URL.createObjectURL(audioBlob);

      soundRef.current = new Howl({
        src: [audioUrl],
        format: ['mp3'],
        onend: () => setIsPlaying(false),
        onplay: () => {
          setIsPlaying(true);
          setIsLoading(false);
        }
      });

      soundRef.current.play();
    } catch (error) {
      console.error('Error generating TTS:', error);
      setIsLoading(false);
    }
  };

  return (
    <button 
      onClick={handlePlay}
      disabled={isLoading}
      className="tts-button"
    >
      {isLoading ? <span>Loading...</span> : isPlaying ? <FaStop /> : <FaPlay />}
    </button>
  );
};

export default AudioPlayer;
```

### Utilisation du Composant
```jsx
<AudioPlayer 
  text="Bonjour, comment allez-vous aujourd'hui ?" 
  language="fr"
/>
```

## Intégration Backend (Spring Boot)
### Configuration
Ajoutez la dépendance dans `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>
```

Configurez les propriétés dans `application.properties`:
```properties
elevenlabs.api.key=your-api-key
elevenlabs.api.url=https://api.elevenlabs.io/v1/text-to-speech
```

### Controller TTS
```java
@RestController
@RequestMapping("/api/tts")
public class TTSController {

    @Value("${elevenlabs.api.key}")
    private String apiKey;
    
    @Value("${elevenlabs.api.url}")
    private String apiUrl;

    @PostMapping("/generate")
    public ResponseEntity<byte[]> generateTTS(
            @RequestBody TTSRequest request,
            @RequestHeader HttpHeaders headers) {
        
        try {
            String voiceId = getVoiceIdForLanguage(request.getLanguage());
            
            String requestJson = String.format(
                "{"text": "%s", "model_id": "eleven_multilingual_v2", "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}}",
                request.getText().replace(""", "\""));
            
            String url = apiUrl + "/" + voiceId;
            HttpPost httpPost = new HttpPost(url);
            httpPost.setHeader("xi-api-key", apiKey);
            httpPost.setHeader("Content-Type", "application/json");
            httpPost.setEntity(new StringEntity(requestJson));
            
            CloseableHttpClient client = HttpClients.createDefault();
            CloseableHttpResponse response = client.execute(httpPost);
            
            byte[] audioBytes = EntityUtils.toByteArray(response.getEntity());
            
            HttpHeaders responseHeaders = new HttpHeaders();
            responseHeaders.setContentType(MediaType.APPLICATION_OCTET_STREAM);
            responseHeaders.setContentLength(audioBytes.length);
            
            return new ResponseEntity<>(audioBytes, responseHeaders, HttpStatus.OK);
            
        } catch (Exception e) {
            return new ResponseEntity<>(HttpStatus.INTERNAL_SERVER_ERROR);
        }
    }
    
    private String getVoiceIdForLanguage(String language) {
        switch (language.toLowerCase()) {
            case "fr":
            case "french":
                return "pFZP5JQG7iQjIQuC4Bku";
            case "en":
            case "english":
                return "9BWtsMINqrJLrRacOk9x";
            case "ar":
            case "arabic":
                return "tavIIPLplRB883FzWU0V";
            default:
                return "pFZP5JQG7iQjIQuC4Bku";
        }
    }
}

class TTSRequest {
    private String text;
    private String language;
    // Getters et Setters
}
```

## Configuration ElevenLabs
- Clé API : Obtenez une clé API depuis le dashboard ElevenLabs
- Voix disponibles :
  - Français : `pFZP5JQG7iQjIQuC4Bku`
  - Anglais : `9BWtsMINqrJLrRacOk9x`
  - Arabe : `tavIIPLplRB883FzWU0V`
- Modèle : `eleven_multilingual_v2`

## Flux de Travail Temps Réel
1. L'utilisateur clique sur l'icône audio dans React
2. Le frontend envoie le texte et la langue au backend
3. Le backend formate la requête pour ElevenLabs
4. Appel à l'API ElevenLabs en temps réel
5. Réception du flux audio et retour au frontend
6. Le frontend joue l'audio immédiatement

## Sécurité et Optimisation
- **Cache** : Implémenter un cache côté serveur
- **Rate limiting** : Limiter les requêtes abusives
- **Sécurité** :
  - Valider le texte
  - Limiter à 5000 caractères
  - Utiliser HTTPS
- **Optimisation** :
  - Pré-génération pour contenu statique
  - Streaming audio pour longs textes

## Dépannage
- **Erreur 401** :
  - Vérifier la clé API
  - Vérifier la configuration
- **Latence** :
  - Optimiser la taille des textes
  - Afficher un loader dans l'UI
- **Problèmes audio** :
  - Vérifier le support MP3 du frontend
  - Vérifier les en-têtes HTTP

## Évolution Future
- Personnalisation des voix
- Paramètres avancés : vitesse, ton, etc.
- Historique des audios
- Intégration TTS locale hors-ligne
