## 1. Glossaire (20 termes)

| #   | Terme                                    | Définition                                                                                                                                                    |
| --- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **LLM** (Large Language Model)           | Modèle de langage entraîné sur de grandes quantités de texte, capable de générer, résumer, traduire ou raisonner en langage naturel.                          |
| 2   | **Token**                                | Unité de base traitée par un LLM — approximativement ¾ d'un mot en anglais. La facturation et les limites de contexte s'expriment en tokens.                  |
| 3   | **Prompt**                               | Instruction ou message envoyé au modèle pour déclencher une génération. Inclut le system prompt et les messages utilisateur.                                  |
| 4   | **System prompt**                        | Instruction de cadrage placée avant la conversation. Définit le rôle, le ton, les contraintes et le format attendu du modèle.                                 |
| 5   | **Contexte (Context window)**            | Fenêtre de tokens visible par le modèle lors d'une inférence. Tout ce qui dépasse cette limite est ignoré.                                                    |
| 6   | **Inférence**                            | Phase d'exécution du modèle : le modèle reçoit un prompt et produit une complétion. Distinct de l'entraînement.                                               |
| 7   | **Temperature**                          | Paramètre (0–1+) contrôlant l'aléatoire de la génération. Proche de 0 → réponses déterministes ; proche de 1 → réponses créatives et variées.                 |
| 8   | **Top-p (nucleus sampling)**             | Paramètre complémentaire à la temperature qui limite le choix des tokens suivants aux plus probables jusqu'à une probabilité cumulée p.                       |
| 9   | **Hallucination**                        | Phénomène par lequel le modèle génère des informations fausses ou inventées présentées avec assurance. Risque majeur en production.                           |
| 10  | **Structured output**                    | Mode dans lequel le modèle est contraint de répondre dans un format prédéfini (JSON, XML…), validable par un schéma.                                          |
| 11  | **Function calling / Tool use**          | Capacité du modèle à décider d'appeler une fonction externe (API, BDD…) et à en intégrer le résultat dans sa réponse.                                         |
| 12  | **RAG** (Retrieval-Augmented Generation) | Architecture combinant une recherche documentaire (vectorielle ou lexicale) avec la génération : les documents pertinents sont injectés dans le prompt.       |
| 13  | **Embedding**                            | Représentation vectorielle dense d'un texte, utilisée pour mesurer la similarité sémantique. Base des systèmes de recherche vectorielle.                      |
| 14  | **Vector store**                         | Base de données spécialisée pour stocker et interroger des embeddings (ex. : Pinecone, Qdrant, pgvector). Composant clé du RAG.                               |
| 15  | **Chunking**                             | Découpage d'un document long en fragments (chunks) avant vectorisation. La taille et le chevauchement des chunks impactent la qualité du RAG.                 |
| 16  | **Fine-tuning**                          | Ré-entraînement partiel d'un modèle sur un jeu de données spécifique pour spécialiser son comportement ou son style.                                          |
| 17  | **Latence**                              | Temps écoulé entre l'envoi du prompt et la réception de la réponse complète. Dépend de la taille du modèle, du nombre de tokens et de l'infrastructure.       |
| 18  | **Streaming**                            | Mode de transmission dans lequel les tokens sont envoyés au client au fur et à mesure de leur génération, réduisant la latence perçue.                        |
| 19  | **Grounding**                            | Fait d'ancrer la réponse du modèle dans des sources vérifiables (documents, données factuelles) pour réduire les hallucinations.                              |
| 20  | **Guardrail**                            | Mécanisme de contrôle (pré ou post-génération) visant à détecter et bloquer des sorties non conformes : contenu nuisible, format invalide, données sensibles. |

---

## 2. Quand utiliser quoi ?

| Mode                            | Quand l'utiliser                                                              | Exemples typiques                                          | À éviter si…                                                                                                                        |
| ------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **Chat (texte libre)**          | Réponse conversationnelle, explication, rédaction, résumé                     | Assistant client, génération de contenu, brainstorming     | La sortie doit être parsée programmatiquement                                                                                       |
| **Structured output (JSON)**    | La réponse alimente directement un système (UI, BDD, API)                     | Extraction d'entités, classification, formulaires, scoring | Le résultat est destiné à être lu uniquement par un humain                                                                          |
| **Tool use (function calling)** | Le modèle doit agir sur le monde réel ou consulter des données dynamiques     | Réservation, requête BDD, envoi d'e-mail, calcul précis    | Les outils disponibles sont trop nombreux ou mal définis                                                                            |
| **RAG (documents)**             | Les réponses doivent se baser sur une base de connaissances privée ou récente | FAQ interne, support technique, analyse de contrats        | Les documents sont trop volumineux pour être chunked correctement ou les questions nécessitent un raisonnement multi-sauts complexe |

---

## 3. Les 4 modes expliqués

### 3.1 Demander du texte (Chat)

Le modèle reçoit un prompt et génère une réponse en langage naturel, sans contrainte de format.

**Flux :**

```
Prompt (string) → LLM → Réponse (string)
```

**Avantages :** Simple, flexible, idéal pour la communication humaine.  
**Limite :** La sortie est imprévisible dans sa structure — impossible à parser de manière fiable.

**Exemple :**

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "Explique le RAG en 3 phrases."}]
)
print(response.content[0].text)  # Texte libre
```

---

### 3.2 Demander un objet JSON (Structured output)

Le modèle est instruit — via le system prompt et/ou un schéma — de retourner exclusivement un objet JSON conforme à une structure attendue.

**Flux :**

```
Prompt + Schéma JSON → LLM → JSON (parseable) → Validation → Objet typé
```

**Avantages :** Intégration directe avec des APIs, bases de données, interfaces. Zéro parsing fragile.  
**Limite :** Le modèle peut encore produire un JSON invalide si le schéma est mal défini ou le prompt trop ambigu.

**Exemple :**

```python
system = """
Tu extrais des informations. Réponds UNIQUEMENT avec un JSON valide,
sans texte autour, respectant ce schéma :
{"nom": string, "email": string, "score": number}
"""
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    system=system,
    messages=[{"role": "user", "content": "Jean Dupont, jean@ex.com, score 8/10"}]
)
data = json.loads(response.content[0].text)
```

---

### 3.3 Donner des outils au modèle (Tool use / Function calling)

Le modèle reçoit la définition de fonctions externes. Il décide lui-même, au cours du raisonnement, quelles fonctions appeler et avec quels arguments. L'application exécute les appels et retourne les résultats au modèle, qui finalise sa réponse.

**Flux :**

```
Prompt + Définitions d'outils → LLM → [tool_use block] → App exécute → [tool_result] → LLM → Réponse finale
```

**Avantages :** Le modèle peut agir (créer, modifier, interroger) sur des systèmes tiers. Il gère lui-même l'orchestration.  
**Limite :** Latence accrue (plusieurs allers-retours), risque d'appels incorrects si les outils sont mal documentés.

**Exemple :**

```python
tools = [{
    "name": "get_weather",
    "description": "Retourne la météo actuelle pour une ville.",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
    }
}]
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=tools,
    messages=[{"role": "user", "content": "Quel temps fait-il à Lyon ?"}]
)
# Le modèle retourne un bloc tool_use → l'app appelle get_weather("Lyon")
```

---

### 3.4 Brancher le modèle à des documents (RAG)

La réponse du modèle est ancrée dans une base documentaire. Un pipeline de retrieval sélectionne les passages les plus pertinents et les injecte dans le contexte avant génération.

**Flux :**

```
Question → Embedding → Recherche vectorielle → Top-k chunks
    → Prompt enrichi (question + chunks) → LLM → Réponse sourcée
```

**Avantages :** Le modèle répond à partir de données privées, récentes ou volumineuses qui ne tiennent pas en contexte. Réponses vérifiables et citables.  
**Limite :** La qualité dépend fortement du retrieval — un mauvais chunking ou une mauvaise requête produit un contexte hors-sujet. Ne remplace pas le fine-tuning pour le comportement.

**Exemple :**

```python
# 1. Retrieval
chunks = vector_store.search(query=user_question, top_k=5)
context = "\n\n".join(chunk.text for chunk in chunks)

# 2. Génération augmentée
prompt = f"""Réponds uniquement à partir des extraits suivants :

{context}

Question : {user_question}"""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": prompt}]
)
```

---

## 4. Les 5 risques classiques

### ⚠️ Risque 1 — Hallucinations

**Description :** Le modèle génère des faits, chiffres, citations ou références qui n'existent pas, avec un ton assuré.

**Pourquoi ça arrive :** Le LLM optimise la vraisemblance statistique du texte, non sa véracité factuelle. En l'absence d'information, il "comble".

**Symptômes :** Noms propres incorrects, dates inventées, articles scientifiques qui n'existent pas.

**Mitigation :**

- Utiliser le **RAG** pour ancrer les réponses dans des sources.
- Demander au modèle de répondre "je ne sais pas" si non-confiant (instruction explicite).
- Mettre en place une étape de **vérification post-génération** (autre appel LLM ou outil de fact-checking).
- Baisser la **temperature** pour les tâches factuelles.

---

### ⚠️ Risque 2 — Format invalide

**Description :** Le modèle retourne un JSON malformé, un schéma incomplet ou insère du texte autour du bloc attendu, cassant le parsing en production.

**Pourquoi ça arrive :** Le modèle est un générateur de texte — il peut ajouter une phrase d'introduction, omettre une virgule ou échapper incorrectement des caractères.

**Symptômes :** `JSONDecodeError`, champs manquants, types inattendus.

**Mitigation :**

- Utiliser le **mode JSON natif** de l'API quand disponible (force le format côté serveur).
- Définir un schéma JSON strict dans le system prompt avec un exemple concret.
- Implémenter un **retry avec correction** : renvoyer l'erreur de parsing au modèle pour qu'il se corrige.
- Valider avec `jsonschema` ou Pydantic avant d'utiliser les données.

---

### ⚠️ Risque 3 — Contexte insuffisant

**Description :** Le modèle ne dispose pas des informations nécessaires pour répondre correctement — soit parce qu'elles dépassent la fenêtre de contexte, soit parce qu'elles n't ont pas été injectées.

**Pourquoi ça arrive :** La context window est limitée. Au-delà, les tokens sont tronqués silencieusement. Le modèle répond quand même, mais sur la base d'informations partielles.

**Symptômes :** Réponses génériques, ignorance de documents pourtant fournis, incohérences dans les conversations longues.

**Mitigation :**

- Mettre en œuvre le **RAG** pour ne charger que les chunks pertinents.
- Résumer ou compresser l'historique de conversation (summarization memory).
- Surveiller l'usage de tokens et alerter avant la limite.
- Utiliser des modèles à large contexte pour les cas critiques.

---

### ⚠️ Risque 4 — Coût non maîtrisé

**Description :** Les coûts d'API explosent en production à cause de prompts surdimensionnés, de boucles d'agents incontrôlées ou d'un usage non anticipé.

**Pourquoi ça arrive :** Chaque token (input et output) est facturé. Un RAG qui injecte 10 000 tokens par requête multiplie les coûts. Un agent qui boucle peut enchaîner des dizaines d'appels.

**Symptômes :** Facture mensuelle imprévisible, timeout budgétaire.

**Mitigation :**

- Définir un **max_tokens** strict sur les outputs.
- Limiter le nombre d'itérations dans les boucles agentiques.
- Choisir le modèle le moins puissant satisfaisant le cas d'usage (ex. Haiku plutôt qu'Opus pour la classification simple).
- Implémenter un **cache sémantique** pour les requêtes répétitives.
- Mettre en place des alertes de budget sur le tableau de bord API.

---

### ⚠️ Risque 5 — Latence

**Description :** Le temps de réponse est trop élevé pour l'expérience utilisateur ou les SLA du système (> 2–5 s pour une interaction chat, > 500 ms pour une API synchrone).

**Pourquoi ça arrive :** La latence d'inférence croît avec la taille du modèle, le nombre de tokens en entrée et la longueur de la réponse. Les boucles tool use ajoutent des allers-retours réseau.

**Symptômes :** UX dégradée, timeouts, abandon utilisateur.

**Mitigation :**

- Activer le **streaming** pour afficher les tokens dès leur production.
- Réduire la taille du prompt (moins de contexte = plus rapide).
- Utiliser un modèle plus léger ou distillé pour les cas temps-réel.
- Paralléliser les appels indépendants (ex. récupération de plusieurs sources en simultané).
- Mettre en cache les réponses aux questions fréquentes (cache exact ou sémantique).

---

## Récapitulatif rapide

```
Besoin d'une réponse lisible par un humain ?         → Chat (texte libre)
Besoin d'alimenter un système programmatique ?       → Structured output (JSON)
Besoin d'agir sur des services tiers ?               → Tool use (function calling)
Besoin de répondre depuis une base documentaire ?    → RAG
```

> **Règle d'or :** commencer simple (chat), structurer quand c'est nécessaire (JSON), agir quand inévitable (tools), ancrer quand les données ne tiennent pas en contexte (RAG). Combiner avec parcimonie.
