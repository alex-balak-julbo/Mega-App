# Configuration d'un Hub Multi-Applications Flask sous Windows

Ce guide explique comment organiser, fusionner et lancer automatiquement plusieurs applications Flask indépendantes (`Invoice`, `Stocks`, `Reports`) sous une seule adresse locale de type `127.0.0.1/stocks`.

---

## 1. Structure des dossiers

Chaque application possède son propre dossier indépendant. Un dossier central nommé `Serveur_Principal` (le Hub) est créé à côté pour orchestrer l'ensemble et héberger l'environnement virtuel.

```text
📁 MesProjetsPython/
│── 📁 Projet_Invoice/
│   └── 📄 app.py          (Contient l'app Flask Invoice)
│── 📁 Projet_Stocks/
│   └── 📄 app.py          (Contient l'app Flask Stocks)
│── 📁 Projet_Reports/
│   └── 📄 app.py          (Contient l'app Flask Reports)
└── 📁 Serveur_Principal/
    ├── 📁 .venv/          (Environnement virtuel global)
    ├── 📄 wsgi.py         (Le script de fusion WSGI)
    └── 📄 lancer.bat      (Le script de démarrage Windows)
```

---

## 2. Configuration de l'environnement virtuel (`venv`)

Pour isoler les dépendances et garantir un lancement sans erreur au démarrage du PC, créez un environnement virtuel unique dans le dossier `Serveur_Principal`.

Ouvrez un terminal (Invite de commandes) et exécutez :

```cmd
cd C:\Chemin\Vers\MesProjetsPython\Serveur_Principal
python -m venv .venv
.venv\Scripts\activate
pip install flask waitress werkzeug
```

---

## 3. Le script de fusion WSGI (`Serveur_Principal/wsgi.py`)

Ce script charge dynamiquement chaque application depuis son dossier respectif en évitant les conflits de nommage (puisqu'elles s'appellent toutes `app.py`), puis utilise `DispatcherMiddleware` pour les lier à un préfixe d'URL.

```python
import sys
import os
import importlib.util
from werkzeug.middleware.dispatcher import DispatcherMiddleware
from flask import Flask
from waitress import serve

# 1. Détermination du dossier parent contenant tous les projets
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# 2. Fonction pour charger proprement l'objet 'app' de chaque sous-dossier
def charger_app(nom_dossier):
    chemin = os.path.join(BASE_DIR, nom_dossier, 'app.py')
    spec = importlib.util.spec_from_file_location("app", chemin)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module.app

# Chargement de chaque mini-app
invoice_app = charger_app('Projet_Invoice')
stocks_app = charger_app('Projet_Stocks')
reports_app = charger_app('Projet_Reports')

# 3. Création de l'application racine (http://127.0.0)
main_app = Flask(__name__)

@main_app.route('/')
def home():
    return "Portail principal : accédez à /invoice, /stocks, ou /reports"

# 4. Fusion des applications avec le Dispatcher
application = DispatcherMiddleware(main_app, {
    '/invoice': invoice_app,
    '/stocks': stocks_app,
    '/reports': reports_app
})

# 5. Lancement avec le serveur WSGI Waitress (robuste sous Windows)
if __name__ == '__main__':
    print("Serveur multi-projets lancé sur http://127.0.0.1:8080")
    serve(application, host='127.0.0.1', port=8080)
```

---

## 4. Script de démarrage automatique (`lancer.bat`)

Ce script active automatiquement l'environnement virtuel `.venv` puis lance le serveur WSGI en arrière-plan.

Créer un fichier `lancer.bat` dans `Serveur_Principal` avec le contenu suivant :

```batch
@echo off
:: 1. Se déplacer dans le dossier du serveur
cd /d "C:\Chemin\Vers\MesProjetsPython\Serveur_Principal"

:: 2. Activer l'environnement virtuel (.venv)
call .venv\Scripts\activate

:: 3. Lancer le serveur WSGI en arrière-plan
start /b python wsgi.py
```

### Exécution automatique au démarrage de Windows :
1. Appuyez sur `Windows + R`, tapez `shell:startup` et validez.
2. Copiez ou créez un raccourci de votre fichier `lancer.bat` à l'intérieur du dossier qui vient de s'ouvrir.

---

## 💡 Règle d'or pour le développement des mini-apps

Le `DispatcherMiddleware` injecte de façon transparente le préfixe de l'URL (`/stocks`, `/reports`, etc.) dans l'environnement de l'application Flask correspondante. 

Pour éviter les erreurs de liens ou de chargement de ressources lors de la navigation :
* **À bannir :** L'écriture de chemins fixes en dur dans vos fichiers HTML (ex: `<a href="/index">`).
* **À utiliser obligatoirement :** La fonction `url_for()` de Flask (ex: `<a href="{{ url_for('index') }}">`). Flask se chargera d'ajouter automatiquement le bon préfixe de dossier (`/stocks/index`) de manière dynamique.
