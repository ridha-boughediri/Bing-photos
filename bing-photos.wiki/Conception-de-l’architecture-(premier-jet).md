1. Infrastructure 
```mermaid
graph TD
    subgraph Infrastructure
        subgraph API_Gateway
            API_Service[API Gateway]
        end
        
        subgraph Services
            GalleryService[Gallery Service]
            S3API[API S3 like Go]
            AuthService[Service d'Authentification]
        end
        
        subgraph Storage
            FileSystem[Filesystem Local]
        end
        
        subgraph Databases
            GalleryDB[(Base de Données GalleryService)]
            AuthDB[(Base de Données AuthService)]
        end
    end

    API_Service -->|Upload, Download| GalleryService
    GalleryService -->|Stockage de Fichiers| S3API
    S3API -->|Sauvegarde des Fichiers| FileSystem
    GalleryService -->|Gestion des Métadonnées| S3API
    
    API_Service -->|Authentification| AuthService
    AuthService -->|Stockage des Utilisateurs| AuthDB
```
1. API Gateway

Description : L'API Gateway sert de point d'entrée unique pour toutes les requêtes venant des utilisateurs. Elle gère le routage des requêtes vers les différents services et applique les règles de sécurité, comme la vérification des tokens d'authentification.

Rôle : Routage des requêtes vers les microservices appropriés (GalleryService, AuthService, ApiS3).
Vérification de l'authentification et gestion des accès aux services protégés.
Aggrégation des réponses de plusieurs services si nécessaire.

```mermaid
graph TD
    User(User) -->|Fait des requêtes| API_Gateway[API Gateway]

    API_Gateway -->|Redirige pour l'authentification| AuthService[AuthService]
    API_Gateway -->|Redirige pour la gestion des galeries| GalleryService[GalleryService]
        AuthService -->|Accède à la base de données| AuthDB[(Base de données Auth)]

    GalleryService -->|Stocke les fichiers| S3Service
    GalleryService -->|Accède à la base de données| GalleryDB[(Base de données Gallery)]

S3Service-->|Accède au file système| FileSysteme[(FileSysteme)]
```
2. AuthService (Service d'Authentification)

Description : Ce microservice gère l'authentification des utilisateurs, y compris l'enregistrement, la connexion, et la gestion des tokens d'authentification.

Rôle : Gérer l'inscription et la connexion des utilisateurs.
Générer et valider les tokens JWT pour l'authentification.
Stocker les informations des utilisateurs dans la base de données associée (AuthDB).

Base de Données (AuthDB) : Contient les informations des utilisateurs, telles que les identifiants et les données d'authentification.

```mermaid
erDiagram
    USER {
        int id PK
        string username
        string password
        string firstName
        string lastName
        string email
        string googleId
        string phoneNumber
        string resetToken
        date birthDate
        date createdAt
        date updatedAt
    }
```
```mermaid
classDiagram
    class User {
      +int id
      +string username
      +string password
      +string firstname
      +string lastname
      +string email
      +string googleId
      +string phoneNumber
      +date birthDate
      +string resetToken
      +date createdAt
      +date updatedAt
      +validatePassword()
    }

    class JWTService {
      +string token
      +datetime expiration
      +datetime issuedAt
      +NewJWTService()
      +generateToken()
      +verifyToken()
    }

    class AuthService {
      +Initialize()
      +loginWithEmail(user: User)
      +registerWithEmail(user: User)
      +forgotPassword(email: string)
      +resetPassword(token: string, newPassword: string)
      +loginWithGoogle()
      +validateToken(token: JWT)
      +generatePasswordResetToken()
      +logout()
    }

    class EmailService {
      +NewEmailService()
      +sendEmailVerification(email: string)
      +sendPasswordResetEmail(email: string)
    }

    class GoogleAuth {
      +NewGoogleAuthService()
      +authenticateWithGoogle()
      +getGoogleUserProfile()
    }

    class DBManager {
      +NewDBManager()
      +beginTransaction() 
      +closeConnection() 
      +autoMigrate()
    }

    class SecurityService {
      +NewSecurityService()
      +hashPassword(password: string)
      +comparePassword(hashedPassword: string, password: string)
      +generateSecureToken()
      +GeneratePasswordResetLink(email: string)
    }

    class Logger {
      +NewLoggerService()
      +logInfo(message: string)
      +logError(message: string)
      +logWarning(message: string)
    }

    %% Relations
    AuthService --> User : manages
    AuthService --> JWTService : handles
    AuthService --> EmailService : uses
    AuthService --> GoogleAuth : integrates with
    AuthService --> DBManager : interacts with
    AuthService --> SecurityService : uses for password hashing
    AuthService --> Logger : logs activity
    DBManager --> User : stores
    EmailService --> Logger : logs email activities
    GoogleAuth --> Logger : logs Google auth attempts

```
Diagramme de séquence login

```mermaid
sequenceDiagram
    participant Client
    participant API_Gateway
    participant AuthService
    participant DB as Database

    Client->>API_Gateway: POST /login
    API_Gateway->>AuthService: Redirige la requête /login
    AuthService->>DB: Vérifie les informations d'utilisateur
    DB-->>AuthService: Retourne les infos utilisateur
    AuthService->>AuthService: Génère JWT
    AuthService-->>API_Gateway: Retourne le token JWT
    API_Gateway-->>Client: Retourne JWT au client
```

3. GalleryService

Description : Le GalleryService gère les opérations sur les fichiers multimédia (photos, vidéos) ainsi que les métadonnées associées. Il permet de créer, lire, mettre à jour, et supprimer les fichiers et les albums, ainsi que de gérer les droits d'accès.

Rôle : Gérer les albums, les fichiers, et les métadonnées associées.
Interagir avec l'API S3-like pour stocker et récupérer les fichiers.
Stocker les métadonnées des fichiers dans la base de données associée (GalleryDB).
Base de Données (GalleryDB) : Stocke les informations sur les fichiers, les albums, les droits d'accès et autres métadonnées.

```mermaid
classDiagram
    class User {
      +int id
      +string username
    }

    class Album {
      +int id
      +string name
      +int userId
      +string mediaPath
      +string type
    }

    class Media {
      +int id
      +int albumId
      +string path
      +string name
      +string type
      +bool isFavorite
      +string hash
    }

    class Access {
      +int id
      +int mediaId
      +int userAccessId
      +datetime expirationDate
      +string code
      +string status
      +string link
    }

    class SimilarGroup {
      +int id
      +int userId
      +datetime createdAt
    }

    class SimilarMedia {
      +int id
      +int similarGroupId
      +int mediaId
      +float similarityScore
    }

    class UserAccess {
      +int id
      +string name
      +int userId
    }

    %% Relations
    User "1" -- "many" Album : "Possède"
    Album "many" -- "many" Media : "Contient"
    Media "1" -- "many" Access : "Est partagé avec"
    UserAccess "many" -- "1" Access : "Accède"
    SimilarGroup "1" -- "many" SimilarMedia : "Regroupe"
    Media "1" -- "many" SimilarMedia : "Est comparé"
```
Diagramme de séquence lien

```mermaid
sequenceDiagram
    participant User as Utilisateur
    participant API_Gateway
    participant GalleryService
    participant DB as Base de Données
    participant SharedUser as Utilisateur partagé

    User->>API_Gateway: Requête pour partager un media/album
    API_Gateway->>GalleryService: Générer un lien unique
    GalleryService->>DB: Enregistrer les détails du lien partagé
    DB-->>GalleryService: Confirmation d'enregistrement
    GalleryService-->>API_Gateway: Lien unique généré
    API_Gateway-->>User: Retour du lien unique
    
    %% Accès via le lien partagé
    SharedUser->>API_Gateway: Accéder au lien unique
    API_Gateway->>GalleryService: Vérifier l'existence du lien
    GalleryService->>DB: Vérifier le lien partagé
    DB-->>GalleryService: Lien valide
    GalleryService-->>API_Gateway: Accès autorisé au contenu
    API_Gateway-->>SharedUser: Contenu accessible
```
Diagramme de séquence code

```mermaid
sequenceDiagram
    participant User as Utilisateur
    participant API_Gateway
    participant GalleryService
    participant DB as Base de Données

    %% Configuration d'un code d'accès pour un album ou un média
    User->>API_Gateway: Configurer un code d'accès pour un album/média
    API_Gateway->>GalleryService: Ajouter un code d'accès à l'album/média
    GalleryService->>DB: Enregistrer le code d'accès dans Access (associer à l'album ou au média)
    DB-->>GalleryService: Confirmation d'enregistrement
    GalleryService-->>API_Gateway: Code d'accès ajouté
    API_Gateway-->>User: Confirmation de la sécurisation

    %% Accès à un album/média sécurisé
    User->>API_Gateway: Accéder à un album/média protégé
    API_Gateway->>GalleryService: Demande de consulter l'album/média protégé
    GalleryService->>DB: Vérifier si un code d'accès est requis
    DB-->>GalleryService: Un code est requis pour l'album/média
    GalleryService-->>API_Gateway: Demander le code d'accès
    API_Gateway-->>User: Saisir le code d'accès
    User->>API_Gateway: Saisie du code d'accès
    API_Gateway->>GalleryService: Validation du code d'accès
    GalleryService->>DB: Vérifier le code d'accès dans Access
    DB-->>GalleryService: Code valide
    GalleryService-->>API_Gateway: Accès autorisé à l'album/média
    API_Gateway-->>User: Accès au contenu
```
4. API S3-like (Go)

Description : C'est un service de gestion de fichiers qui imite les fonctionnalités d'un service de stockage de type S3. Il permet de gérer les fichiers et dossiers via un système de fichiers local.

Rôle : Gérer le stockage physique des fichiers ( list,upload, download, suppression).
Manipuler les dossiers et les fichiers via le système de fichiers local.
Fournir une interface pour les autres services (comme GalleryService) pour interagir avec les fichiers.

Diagramme de classe

```mermaid
classDiagram
    class BucketHandler {
        -Storage storage.Storage
        +HandleListBuckets(http.ResponseWriter, *http.Request)
        +HandleCreateBucket(http.ResponseWriter, *http.Request)
        +HandleGetBucket(http.ResponseWriter, *http.Request)
        +HandleDeleteBucket(http.ResponseWriter, *http.Request)
    }

    class ObjectHandler {
        -Storage storage.Storage
        +HandleAddObject(http.ResponseWriter, *http.Request)
        +HandleCheckObjectExist(http.ResponseWriter, *http.Request)
        +HandleDownloadObject(http.ResponseWriter, *http.Request)
        +HandleListObjects(http.ResponseWriter, *http.Request)
        +HandleDeleteObject(http.ResponseWriter, *http.Request)
    }

    class Storage {
        <<interface>>
        +ListBuckets() []BucketDTO
        +CreateBucket(bucketName string) error
        +CheckBucketExists(bucketName string) (bool, error)
        +AddObject(bucketName string, objectName string, data io.Reader, checksum string) error
        +CheckObjectExist(bucketName string, objectName string) (bool, time.Time, int64, error)
        +GetObject(bucketName string, objectName string) ([]byte, os.FileInfo, error)
        +ListObjects(bucketName string, prefix string, marker string, maxKeys int) ([]ObjectDTO, error)
        +DeleteBucket(bucketName string) error
    }

    class FileStorage {
        +ListBuckets() []BucketDTO
        +CreateBucket(bucketName string) error
        +CheckBucketExists(bucketName string) (bool, error)
        +AddObject(bucketName string, objectName string, data io.Reader, checksum string) error
        +CheckObjectExist(bucketName string, objectName string) (bool, time.Time, int64, error)
        +GetObject(bucketName string, objectName string) ([]byte, os.FileInfo, error)
        +ListObjects(bucketName string, prefix string, marker string, maxKeys int) ([]ObjectDTO, error)
        +DeleteBucket(bucketName string) error
    }

    class DTO {
        <<abstract>>
    }

    class BucketDTO {
        +string Name
        +time.Time CreationDate
        +string LocationConstraint
        +bool LockConfig
        +bool Delimiter
    }

    class ObjectDTO {
        +string Key
        +int64 Size
        +time.Time LastModified
        +string ContentType
        +map<string, string> Metadata
    }

    class ListAllMyBucketsResult {
        +[]BucketDTO Buckets
    }

    class DeleteResult {
        +[]Deleted DeletedResult
    }

    class Deleted {
        +string Key
    }

    class MetadataManager {
        +AddMetadata(objectKey string, metadata map[string, string>)
        +GetMetadata(objectKey string) map[string, string>
        +DeleteMetadata(objectKey string)
    }

    DTO <|-- BucketDTO
    DTO <|-- ObjectDTO
    DTO <|-- ListAllMyBucketsResult
    DTO <|-- DeleteResult
    DeleteResult --> Deleted
    Storage <|.. FileStorage
    BucketHandler --> Storage
    ObjectHandler --> Storage
    BucketHandler --> DTO
    ObjectHandler --> DTO
    ObjectDTO --> MetadataManager : "Utilise pour gérer les métadonnées"
```

Diagramme de séquence upload

``` mermaid
sequenceDiagram
    participant User as Utilisateur
    participant API_Gateway as API Gateway
    participant GalleryService as GalleryService
    participant S3API as API S3-like
    participant FileSystem as Système de fichiers
    participant DB as Base de Données

    User->>API_Gateway: Requête POST /media/upload (fichier + token JWT)
    API_Gateway->>API_Gateway: Vérifier le token JWT
    alt Authentification réussie
        API_Gateway->>GalleryService: Transmettre le fichier et les métadonnées
        GalleryService->>S3API: Requête d'upload du fichier
        S3API->>FileSystem: Enregistrer le fichier dans le système de fichiers
        FileSystem-->>S3API: Confirmation de stockage
        S3API-->>GalleryService: Confirmation d'upload réussi
        
        %% Enregistrement des métadonnées
        GalleryService->>DB: Enregistrer les métadonnées du fichier
        DB-->>GalleryService: Confirmation d'enregistrement
        
        %% Réponse à l'utilisateur
        GalleryService->>API_Gateway: Confirmation d'upload réussi
        API_Gateway-->>User: Réponse avec confirmation (200 OK)
    else Authentification échouée
        API_Gateway-->>User: Réponse d'erreur (401 Unauthorized)
    end
```

Diagramme de séquence killerFeature photos similaires 

```mermaid
sequenceDiagram
    participant User as Utilisateur
    participant FrontEnd as Interface Utilisateur
    participant API_Gateway as API Gateway
    participant GalleryService as Gallery Service
    participant DB as PostgreSQL
    participant BackgroundJob as Tâche de Fond

    User->>FrontEnd: Clique sur le bouton "Détecter les photos similaires"
    FrontEnd->>API_Gateway: Envoie une requête de détection
    API_Gateway->>GalleryService: Transmet la demande de détection
    GalleryService->>DB: Récupère les pHash des photos de l'utilisateur
    GalleryService->>BackgroundJob: Déclenche la comparaison des photos
    BackgroundJob->>DB: Accède aux pHash pour trouver les correspondances
    BackgroundJob->>GalleryService: Identifie les groupes de photos similaires
    GalleryService->>DB: Met à jour les informations de groupes de photos
    GalleryService->>API_Gateway: Répond avec les résultats de la détection
    API_Gateway->>FrontEnd: Retourne les résultats à l'interface utilisateur
    FrontEnd->>User: Affiche les groupes de photos similaires détectés
```

5. Système de Fichiers (Filesystem Local)

Description : Le système de fichiers local est utilisé pour stocker physiquement les fichiers. Les données binaires (photos, vidéos) sont enregistrées et récupérées ici via l'API S3-like.

Rôle : Héberger les fichiers uploadés par les utilisateurs.
Servir de stockage persistant pour les données binaires.

Diagramme de séquence de la création d'un bucket

```mermaid
sequenceDiagram
    participant User as Utilisateur
    participant API_Gateway as API Gateway
    participant GalleryService as Service de Galerie
    participant S3API as API S3-like
    participant FileSystem as Système de Fichiers

    User ->> API_Gateway: Demande de création d'un nouveau bucket (nom du bucket)
    API_Gateway ->> GalleryService: Vérifier si le nom du bucket existe dans la base de données
    GalleryService ->> GalleryService: Requête à la base de données pour vérifier l'existence du bucket
    alt Le bucket existe
        GalleryService -->> API_Gateway: Retourner une erreur (le bucket existe déjà)
        API_Gateway -->> User: Échec de la création du bucket (existe déjà)
    else Le bucket n'existe pas
        GalleryService ->> S3API: Procéder à la création d'un nouveau bucket
        S3API ->> FileSystem: Créer un répertoire pour le nouveau bucket
        FileSystem -->> S3API: Confirmation de la création du répertoire
        S3API ->> GalleryService: Informer de la création réussie du répertoire
        GalleryService ->> GalleryService: Insérer les métadonnées du nouveau bucket dans la base de données
        GalleryService -->> API_Gateway: Création du bucket réussie
        API_Gateway -->> User: Bucket créé avec succès
    end

```

Fonctionnement Global :

Authentification et Sécurité : Les utilisateurs s'authentifient via l'AuthService, qui gère les tokens JWT. L'API Gateway vérifie les tokens avant de permettre l'accès aux services protégés.

Gestion des Fichiers : Les utilisateurs peuvent uploader des fichiers via l'API Gateway, qui transmet les requêtes au GalleryService. Le GalleryService utilise l'API S3-like pour gérer le stockage physique des fichiers.

Stockage des Métadonnées : Les métadonnées liées aux fichiers (comme le chemin, les droits d'accès, les informations d'album) sont stockées dans le GalleryDB.

Interaction entre les Services : L'API Gateway assure la communication entre les différents services, routant les requêtes et appliquant les règles de sécurité.





