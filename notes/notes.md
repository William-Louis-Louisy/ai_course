## **Le cœur du système**

### **Modèle**

C’est un réseau de neurones entraîné sur de grandes quantités de texte, capable de prédire le token suivant le plus probable étant donné une séquence d'entrée. (Un peu comme l’autocomplétion des messages sur smartphone)

Le modèle est un fichier de milliards de paramètres. Il ne "pense" pas, il calcule des distributions de probabilité sur le vocabulaire à chaque étape. GPT-4, Claude, Mistral sont des modèles différents, avec des architectures et des données d'entraînement différentes.

### **Contexte**

La fenêtre de tokens visible par le modèle à un instant donné : tout ce qu'il peut "lire" pour générer sa réponse.

Le contexte contient le prompt système, l'historique de la conversation, les documents injectés, les résultats d'outils, etc. Sa taille est limitée (ex. 200 000 tokens pour Claude). Au-delà de cette limite, les informations les plus anciennes disparaissent. Tout ce qui n'est pas dans le contexte est invisible au modèle.

[ system prompt ] + [ docs injectés ] + [ historique ] + [ message user ] → génération

## **Les entrées**

### **Prompt**

L'instruction fournie au modèle tout texte placé en entrée pour orienter sa génération.

Un prompt peut être un message utilisateur simple, un prompt système (instructions permanentes), ou les deux combinés. La qualité du prompt conditionne directement la qualité de la sortie : être précis, donner du contexte, spécifier le format attendu sont les leviers principaux. On parle de "prompt engineering" pour l'art de bien formuler ces instructions.

Système : "Tu es un assistant RH concis."User : "Rédige un mail de refus pour ce candidat."

## **Les sorties**

### **Sortie libre**

Le modèle génère du texte non contraint : prose, code, dialogue, analyse sans format imposé.

C'est le mode par défaut. Le modèle choisit librement la structure, la longueur, le ton. Adapté aux cas où la lisibilité humaine prime : rédaction, explication, résumé, conversation. L'inconvénient : la sortie est difficilement exploitable par un programme sans post-traitement.

### **Sortie structurée**

Le modèle est contraint à produire une sortie dans un format défini typiquement JSON ou XML directement exploitable par du code.

On définit un schéma (ex. "retourne un objet avec les champs nom, date, montant") et le modèle garantit de s'y conformer. Indispensable dans les pipelines automatisés où la sortie alimente une base de données, un affichage UI, ou un autre système. Les APIs modernes exposent cette contrainte via le paramètre "response_format".

{"nom": "Alice", "date": "2025-04-01", "montant": 1200}

## **Les extensions**

### **Outil**

Une fonction externe que le modèle peut décider d'appeler pour obtenir des informations ou déclencher une action dans le monde réel.

On déclare des outils disponibles (recherche web, lecture de fichier, envoi d'email, requête SQL…). Quand le modèle juge qu'un outil est nécessaire, il émet un appel structuré (nom + paramètres) ; le résultat est injecté dans le contexte, puis la génération reprend. C'est le mécanisme derrière tous les "agents" : le modèle agit, pas seulement répond.

Modèle → appelle search("météo Paris") → reçoit résultat → génère réponse

### **RAG**

Retrieval-Augmented Generation : technique consistant à récupérer des documents pertinents dans une base de connaissances et à les injecter dans le contexte avant la génération.

Le problème résolu : le modèle ne connaît que ce sur quoi il a été entraîné. Avec le RAG, on connecte une base documentaire externe (docs internes, site web, base de connaissances). Une requête de recherche vectorielle trouve les passages les plus proches sémantiquement, qui sont ensuite fournis au modèle comme "documents de référence". Le modèle répond en s'appuyant sur ces données fraîches, réduisant les hallucinations.

Question → recherche vectorielle → top-k passages → [contexte enrichi] → réponse sourcée
