# 🏗️ Architecture et Design du Module Backend LLM

> **Module** : `vibe/core/llm/backend/`  
> **Fork** : `rrobbes/mistral-vibe`  
> **Partie** : 2/13 - Conception et Principes

---

## 📐 Principes de Design

Le module `backend/` a été conçu en suivant des **principes architecturaux solides** qui garantissent sa maintenabilité, son extensibilité et sa robustesse.

### 1. 🎯 Séparation des Préoccupations (Separation of Concerns)

**Principe** : Chaque composant a **une seule responsabilité**, bien définie et claire.

**Application dans le module** :

```
┌─────────────────────────────────────────────────────────────────┐
│  PRINCIPE : Séparation des Préoccupations                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   ADAPTERS          │  │    BACKENDS          │  │      FACTORY         │
│  (Traducteurs)      │  │   (Transporteurs)    │  │    (Créateurs)       │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│  • Conversion       │  │  • Transport HTTP   │  │  • Création des      │
│    des formats       │  │  • Gestion du       │  │    backends          │
│  • Parsing des       │  │    streaming        │  │  • Configuration    │
│    réponses          │  │  • Retry            │  │    des timeouts      │
│  • Gestion des       │  │  • Pacing           │  │  • Lazy loading      │
│    spécificités API  │  │  • Tracing          │  │                     │
│                     │  │                     │  │                     │
│  ❌ Ne gère pas :    │  │  ❌ Ne gère pas :    │  │  ❌ Ne gère pas :    │
│  • Transport         │  │  • Conversion des   │  │  • Transport         │
│  • Retry             │  │    formats          │  │  • Parsing           │
│  • Pacing            │  │  • Parsing          │  │  • Conversion        │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
```

**Avantages** :
- ✅ **Code plus simple** : Chaque classe fait une seule chose
- ✅ **Maintenabilité** : Modifier un aspect ne nécessite pas de toucher aux autres
- ✅ **Testabilité** : Chaque composant peut être testé isolément
- ✅ **Réutilisabilité** : Les composants peuvent être réutilisés dans d'autres contextes

### 2. 🔄 Inversion des Dépendances (Dependency Inversion - Principe D de SOLID)

**Principe** : Les modules de haut niveau **ne devraient pas dépendre** des modules de bas niveau. Les deux devraient dépendre d'**abstractions**.

**Application dans le module** :

```
┌─────────────────────────────────────────────────────────────────┐
│  PRINCIPE : Inversion des Dépendances                            │
└─────────────────────────────────────────────────────────────────┘

SANS INVERSION (❌ Mauvaise pratique) :

┌─────────────────┐       ┌─────────────────┐
│   AgentLoop      │──────▶│   MistralBackend │
│   (Haut niveau)  │       │   (Bas niveau)   │
└─────────────────┘       └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │   Anthropic     │
                        │   Backend        │
                        └─────────────────┘

→ Problème : AgentLoop dépend directement de MistralBackend et AnthropicBackend
→ Changement : Ajouter un nouveau backend = modifier AgentLoop

AVEC INVERSION (✅ Bonne pratique) :

┌─────────────────┐       ┌─────────────────┐
│   AgentLoop      │──────▶│   BackendLike   │
│   (Haut niveau)  │       │   (ABSTRACTION)  │
└─────────────────┘       └────────┬────────┘
                               │
            ┌──────────────────────┼──────────────────┐
            │                      │                  │
            ▼                      ▼                  ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   MistralBackend │  │  GenericBackend  │  │  FakeBackend    │
│   (Implémentation)│  │  (Implémentation)│  │  (Pour les tests)│
└─────────────────┘  └─────────────────┘  └─────────────────┘

→ Solution : AgentLoop dépend de l'abstraction BackendLike
→ Avantage : Ajouter un nouveau backend = implémenter BackendLike
```

**Avantages** :
- ✅ **Flexibilité** : On peut changer d'implémentation sans modifier le code client
- ✅ **Extensibilité** : Ajouter un nouveau backend ne nécessite pas de modifier le code existant
- ✅ **Testabilité** : On peut facilement créer des mocks (FakeBackend)

### 3. 🎭 Pattern Strategy

**Principe** : Définir une famille d'algorithmes, les encapsuler, et les rendre interchangeables.

**Application dans le module** :

Les **adapters** représentent différentes **stratégies** pour communiquer avec les API LLM.

```
┌─────────────────────────────────────────────────────────────────┐
│  PATTERN : Strategy                                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Context (GenericBackend)                      │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  strategy: APIAdapter                                        │ │
│  │                                                             │ │
│  │  def complete_streaming(self, ...):                         │ │
│  │      adapter = self._get_adapter(api_style)  # ← Choix    │ │
│  │      req = adapter.prepare_request(...)       # ← Stratégie│ │
│  │      # ...                                                   │ │
│  │      chunk = adapter.parse_response(...)      # ← Stratégie│ │
│  │      return chunk                                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  OpenAI         │  │  Anthropic      │  │  Vertex         │
│  Strategy        │  │  Strategy        │  │  Strategy        │
│                 │  │                 │  │                 │
│  +prepare_request│  │  +prepare_request│  │  +prepare_request│
│  +parse_response │  │  +parse_response │  │  +parse_response │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

**Avantages** :
- ✅ **Interchangeabilité** : On peut changer d'adapter à l'exécution
- ✅ **Extensibilité** : Ajouter un nouvel adapter = ajouter une nouvelle stratégie
- ✅ **Isolation** : Chaque stratégie est encapsulée dans sa propre classe

### 4. 🏭 Pattern Factory

**Principe** : Fournir une interface pour créer des objets, en laissant les sous-classes décider quelle classe instancier.

**Application dans le module** :

La **factory** centralise la création des backends :

```
┌─────────────────────────────────────────────────────────────────┐
│  PATTERN : Factory                                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Creator (factory.py)                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  BACKEND_FACTORY: dict[Backend, Callable] = {               │ │
│  │      Backend.MISTRAL: _create_mistral_backend,               │ │
│  │      Backend.GENERIC: _create_generic_backend,                │ │
│  │  }                                                           │ │
│  │                                                             │ │
│  │  def create_backend(provider, timeout, ...):                │ │
│  │      backend_type = Backend(provider.backend)                │ │
│  │      factory = BACKEND_FACTORY[backend_type]                 │ │
│  │      return factory(provider=provider, timeout=timeout, ...) │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Concrete      │  │   Concrete      │  │   (À venir)     │
│   Product        │  │   Product        │  │   Concrete      │
│   MistralBackend │  │  GenericBackend  │  │   Product        │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

**Avantages** :
- ✅ **Centralisation** : Toute la logique de création au même endroit
- ✅ **Extensibilité** : Ajouter un backend = ajouter une entrée dans le registry
- ✅ **Consistance** : Tous les backends sont créés avec la même configuration
- ✅ **Lazy Loading** : Les modules sont importés seulement quand nécessaires

---

## 🏗️ Organisation en Couches

Le module est organisé en **quatre couches logiques** distinctes, chacune avec une responsabilité claire :

### 📝 Couche 1 : Interface

**Fichier** : `vibe/core/llm/types.py`  
**Rôle** : Définir le contrat que tous les backends doivent respecter

```
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 1 : INTERFACE                                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  BackendLike (Protocol)                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • __aenter__() : BackendLike                                │ │
│  │  • __aexit__() : None                                        │ │
│  │  • complete() : LLMChunk                                     │ │
│  │  • complete_streaming() : AsyncGenerator[LLMChunk]            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

Fonction :
  - Définir l'interface commune pour tous les backends
  - Permettre le polymorphisme
  - Faciliter le mocking pour les tests
```

### 🔄 Couche 2 : Abstraction

**Fichier** : `vibe/core/llm/backend/base.py`  
**Rôle** : Définir le contrat pour la conversion de formats

```
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 2 : ABSTRACTION                                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APIAdapter (ABC - Abstract Base Class)                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • endpoint : ClassVar[str]                                 │ │
│  │  • prepare_request() : PreparedRequest                      │ │
│  │  • parse_response() : LLMChunk                               │ │
│  │  • parse_stream() : AsyncGenerator[ParsedStreamChunk]      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

Fonction :
  - Définir le contrat pour les adapters
  - Fournir des fonctions utilitaires communes
  - Centraliser la logique de conversion

┌─────────────────────────────────────────────────────────────────┐
│  Types et Classes Utilitaires                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • PreparedRequest (endpoint, headers, body, base_url)      │ │
│  │  • ParsedStreamChunk (data, chunk)                         │ │
│  │  • Fonctions : apply_reasoning_effort(), build_chat_payload()│ │
│  │                 build_auth_headers(), finalize_chat_request() │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### ⚙️ Couche 3 : Implémentation

**Fichiers** : 
- `generic.py` - GenericBackend
- `mistral.py` - MistralBackend
- `anthropic.py` - AnthropicAdapter
- `openai_responses.py` - OpenAIResponsesAdapter
- `reasoning_adapter.py` - ReasoningAdapter
- `vertex.py` - VertexAnthropicAdapter
- `_image.py` - ImageManager

**Rôle** : Implémenter les backends et adapters concrets

```
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 3 : IMPLEMENTATION                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐  ┌─────────────────────────┐
│      BACKENDS            │  │       ADAPTERS          │
├─────────────────────────┤  ├─────────────────────────┤
│  • GenericBackend       │  │  • OpenAIAdapter        │
│    - HTTP Client        │  │  • AnthropicAdapter     │
│    - AdaptivePacer      │  │  • OpenAIResponses      │
│    - Retry Logic        │  │  • ReasoningAdapter     │
│    - Streaming          │  │  • VertexAnthropic      │
│    - OpenTelemetry      │  │                         │
│                         │  │  MAPPERS :              │
│  • MistralBackend       │  │  • MistralMapper        │
│    - Mistral SDK Client │  │  • AnthropicMapper      │
│    - MistralMapper      │  │                         │
│    - Vertex Support     │  │  UTILITIES :            │
└─────────────────────────┘  │  • VertexCredentials     │
                           └─────────────────────────┘
                           
┌─────────────────────────┐
│      IMAGE MODULE        │
│  • to_data_uri()         │
│  • to_base64()           │
│  • Cache LRU             │
└─────────────────────────┘
```

### 🏭 Couche 4 : Factory

**Fichier** : `vibe/core/llm/backend/factory.py`  
**Rôle** : Créer les backends avec la bonne configuration

```
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 4 : FABRIQUE                                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Factory Functions                                               │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  BACKEND_FACTORY: dict[Backend, Callable]                   │ │
│  │  _create_mistral_backend(**kwargs) : BackendLike            │ │
│  │  _create_generic_backend(**kwargs) : BackendLike           │ │
│  │                                                             │ │
│  │  def create_backend(                                          │ │
│  │      provider: ProviderConfig,                                │ │
│  │      timeout: float = DEFAULT_API_TIMEOUT,                 │ │
│  │      retry_max_elapsed_time: float = ... ,                 │ │
│  │      connect_timeout: float = ...,                          │ │
│  │      write_timeout: float = ...,                             │ │
│  │      pool_timeout: float = ...,                              │ │
│  │      enable_otel: bool = False,                              │ │
│  │      on_retry: RetryObserver | None = None,                  │ │
│  │  ) -> BackendLike:                                            │ │
│  │      backend = Backend(provider.backend)                     │ │
│  │      factory = BACKEND_FACTORY[backend]                      │ │
│  │      return factory(...)                                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

Fonction :
  - Centraliser la création des backends
  - Configurer les timeouts et options
  - Gérer le lazy loading des modules
```

---

## 🎯 Pourquoi Cette Architecture ?

Cette organisation en couches offre de **nombreux avantages** :

### 1. ✅ Maintenabilité

- **Code plus simple** : Chaque composant a une responsabilité claire
- **Facile à comprendre** : La séparation des préoccupations rend le code plus lisible
- **Moins d'effets de bord** : Les modifications sont localisées

### 2. ✅ Extensibilité

- **Ajouter un provider** = Créer un nouvel adapter
- **Ajouter un backend** = Ajouter une entrée dans la factory
- **Modifier une API** = Modifier uniquement l'adapter concerné

### 3. ✅ Testabilité

- **Mocking facile** : On peut créer des FakeBackend qui implémentent BackendLike
- **Tests unitaires** : Chaque couche peut être testée isolément
- **Tests d'intégration** : Facile à composer

### 4. ✅ Flexibilité

- **Combiner les composants** : On peut utiliser GenericBackend avec n'importe quel adapter
- **Remplacer des composants** : On peut changer d'adapter sans toucher au backend
- **Personnalisation** : Facile d'étendre ou de modifier le comportement

### 5. ✅ Robustesse

- **Erreurs confinées** : Les erreurs sont gérées au niveau approprié
- **Isolation** : Chaque composant peut être testé et validé séparément
- **Résilience** : Les mécanismes de retry et pacing sont centralisés

### 6. ✅ Performance

- **Lazy loading** : Les modules lourds sont chargés seulement quand nécessaires
- **Pooling de connexions** : Réutilisation des connexions HTTP
- **Caching** : Images et credentials sont cachés

---

## 📊 Comparaison avec d'Autres Architectures

| Architecture | Maintenabilité | Extensibilité | Testabilité | Performance | Complexité |
|--------------|----------------|---------------|-------------|-------------|------------|
| **Monolithique** | ❌ Faible | ❌ Faible | ❌ Faible | ✅ Bonne | ❌ Élevée |
| **Microservices** | ✅ Bonne | ✅ Bonne | ✅ Bonne | ❌ Faible | ❌ Élevée |
| **Couches (Ce Module)** | ✅ Bonne | ✅ Bonne | ✅ Bonne | ✅ Bonne | ⚠️ Moyenne |
| **Plugins** | ✅ Bonne | ✅ Bonne | ✅ Bonne | ⚠️ Moyenne | ⚠️ Moyenne |

**Notre architecture** : **Le meilleur compromis** entre tous ces critères.

---

## 🎓 À Retenir

1. **Séparation des préoccupations** : Chaque couche a une responsabilité unique
2. **Inversion des dépendances** : Les haut niveaux dépendent d'abstractions, pas de concretions
3. **Pattern Strategy** : Les adapters sont des stratégies interchangeables
4. **Pattern Factory** : La création des backends est centralisée
5. **4 couches** : Interface → Abstraction → Implémentation → Factory

**Résultat** : Une architecture **modulaire, extensible, testable et robuste** qui permet à Vibe de supporter de nombreux providers LLM avec un minimum de code dupliqué.

### 📚 Prochaine Étape

Pour comprendre **les fondations** de cette architecture, lisez :
→ **[BackendLike et APIAdapter : Les Fondations](./03-FOUNDATIONS.md)**
