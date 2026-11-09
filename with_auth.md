# Architecture Flask Multi-Apps avec Authentification Centralisée (JWT + Refresh Token)

Ce guide explique comment mettre en place un serveur d'authentification centralisé qui distribue des **Access Tokens** (durée courte) et des **Refresh Tokens** (durée longue), permettant de sécuriser l'accès aux sous-applications (`/stocks`, `/reports`) unifiées via le WSGI Dispatcher.

---

## 1. Installation des dépendances

Activez votre environnement virtuel `.venv` dans `Serveur_Principal` et installez l'extension JWT pour Flask :

```cmd
.venv\Scripts\activate
pip install flask waitress werkzeug flask-jwt-extended
```

---

## 2. Structure des fichiers mise à jour

Le dossier `Serveur_Principal` contient désormais la logique d'authentification globale et les clés secrètes partagées.

```text
📁 MesProjetsPython/
│── 📁 Projet_Invoice/
│   └── 📄 app.py          (Vérifie le JWT pour /invoice)
│── 📁 Projet_Stocks/
│   └── 📄 app.py          (Vérifie le JWT pour /stocks)
└── 📁 Serveur_Principal/
    ├── 📄 wsgi.py         (Gère l'authentification + le Dispatcher)
    └── 📄 lancer.bat
```

---

## 3. Le Serveur d'Authentification Centralisé (`Serveur_Principal/wsgi.py`)

Ce fichier configure **Flask-JWT-Extended**. Il expose les routes `/login` et `/refresh` pour gérer la session utilisateur, puis distribue les requêtes vers les sous-applications.

```python
import sys
import os
import datetime
import importlib.util
from flask import Flask, jsonify, request
from flask_jwt_extended import JWTManager, create_access_token, create_refresh_token, jwt_required, get_jwt_identity
from werkzeug.middleware.dispatcher import DispatcherMiddleware
from waitress import serve

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# --- 1. CONFIGURATION DE L'APPLICATION CENTRALISÉE (AUTH) ---
auth_app = Flask(__name__)

# Configuration JWT (À changer en production)
auth_app.config["JWT_SECRET_KEY"] = "super-cle-secrete-a-changer-12345" 
auth_app.config["JWT_ACCESS_TOKEN_EXPIRES"] = datetime.timedelta(minutes=15) # Durée de vie courte
auth_app.config["JWT_REFRESH_TOKEN_EXPIRES"] = datetime.timedelta(days=30)    # Durée de vie longue

jwt = JWTManager(auth_app)

# Base de données utilisateurs fictive (Simulée)
UTILISATEURS = {"admin": "password123"}

@auth_app.route('/login', methods=['POST'])
def login():
    username = request.json.get("username", None)
    password = request.json.get("password", None)
    
    if username not in UTILISATEURS or UTILISATEURS[username] != password:
        return jsonify({"msg": "Identifiants incorrects"}), 401

    # Génération des deux tokens
    access_token = create_access_token(identity=username)
    refresh_token = create_refresh_token(identity=username)
    
    return jsonify({
        "access_token": access_token,
        "refresh_token": refresh_token
    }), 200

@auth_app.route('/refresh', methods=['POST'])
@jwt_required(refresh=True)
def refresh():
    # Génère un nouvel access token à partir du refresh token valide
    current_user = get_jwt_identity()
    new_access_token = create_access_token(identity=current_user)
    return jsonify({"access_token": new_access_token}), 200

@auth_app.route('/')
def index():
    return jsonify({"msg": "Bienvenue sur le Hub. Connectez-vous via /login"})


# --- 2. CHARGEMENT DYNAMIQUE DES MINI-APPS ---
def charger_app(nom_dossier):
    chemin = os.path.join(BASE_DIR, nom_dossier, 'app.py')
    spec = importlib.util.spec_from_file_location("app", chemin)
    module = importlib.util.module_from_spec(spec)
    
    # Injection de la configuration JWT commune dans chaque sous-application
    module.app.config["JWT_SECRET_KEY"] = auth_app.config["JWT_SECRET_KEY"]
    JWTManager(module.app)
    
    spec.loader.exec_module(module)
    return module.app

invoice_app = charger_app('Projet_Invoice')
stocks_app = charger_app('Projet_Stocks')


# --- 3. WSGI DISPATCHER & SERVEUR ---
application = DispatcherMiddleware(auth_app, {
    '/invoice': invoice_app,
    '/stocks': stocks_app
})

if __name__ == '__main__':
    print("Serveur avec Auth JWT lancé sur http://127.0.0.1:8080")
    serve(application, host='127.0.0.1', port=8080)
```

---

## 4. Sécuriser une Mini-App (`Projet_Stocks/app.py`)

Dans vos sous-projets, vous n'avez pas besoin de reconfigurer JWT à partir de zéro. L'instance `JWTManager` lui a été injectée par le script `wsgi.py`. Il vous suffit d'importer et d'utiliser le décorateur `@jwt_required()`.

```python
from flask import Flask, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity

app = Flask(__name__)

@app.route('/')
@jwt_required() # Bloque l'accès si l'Access Token JWT est manquant ou expiré
def index():
    # Récupère l'identité de l'utilisateur stockée dans le token
    utilisateur_actuel = get_jwt_identity()
    return jsonify({
        "msg": "Accès autorisé à l'application STOCKS",
        "utilisateur": utilisateur_actuel
    }), 200

@app.route('/public')
def public():
    return jsonify({"msg": "Cette route dans Stocks ne nécessite pas d'authentification"})
```

---

## 5. Comment tester votre flux d'authentification

Pour tester cette architecture (par exemple avec un outil comme Postman ou Insomnia) :

1. **Connexion :** Envoyez une requête `POST` sur `http://127.0.0` avec le JSON `{"username": "admin", "password": "password123"}`. Vous recevez un `access_token` et un `refresh_token`.
2. **Accès à une app :** Effectuez une requête `GET` sur `http://127.0.0`. Ajoutez l'Access Token dans les en-têtes HTTP (`Authorization: Bearer <VOTRE_ACCESS_TOKEN>`). Si le token est valide, l'application Stocks répond.
3. **Expiration & Refresh :** Si l'Access Token expire (après 15 minutes), l'application répondra `401 Unauthorized`. Envoyez alors une requête `POST` sur `http://127.0.0` en fournissant le Refresh Token (`Authorization: Bearer <VOTRE_REFRESH_TOKEN>`) pour obtenir instantanément un nouvel Access Token sans demander à l'utilisateur de retaper son mot de passe.
