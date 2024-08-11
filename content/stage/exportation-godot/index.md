+++
title = "Exportation Godot"
+++

## 1. Automatiser l'exportation de projet Godot

La première tâche que j'ai eu lors de mon stage fut d'automatiser le processus d'exportation de projet du moteur Godot.

### 1. Les templates d'exportation

J'ai commencé par utiliser Godot et exporter un projet depuis l'interface graphique du logiciel.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-export-menu.png?raw=true", alt="Godot export menu", no_hover=true)}}

Le fait d'utiliser l'interface m'a permis de comprendre et d'identifier les différents paramètres importants à l'exportation d'un projet.

Notamment lors de la première exportation, si le logiciel ne détecte pas de template d'exportation, il propose d'en télécharger un depuis le miroir de notre choix.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-template-download1.png?raw=true", alt="Godot template download", no_hover=true)}}

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-template-download2.png?raw=true", alt="Godot template download", no_hover=true)}}

Suite à cette découverte, j'ai ajouté le code suivant dans mon script pour vérifier si un template de la bonne version existe déjà. Si ce n'est pas le cas, je le télécharge depuis GitHub.

```bash
# Check if there are already export templates
echo "Check if there are export templates..."

mkdir -p ~/.local/share/godot/export_templates/$version

exportTpl=$(ls ~/.local/share/godot/export_templates/$version)

linkVersion=$(echo $version | grep -oP '(\d+(\.\d+)+)')


# Download export templates if there aren't already downloaded
if [[ $exportTpl == "" ]];
then
	echo "Export templates not found..."
	echo "Downloading export templates..."
	wget -q https://github.com/godotengine/godot/releases/download/$linkVersion-stable/Godot_v$linkVersion-stable_export_templates.tpz
	unzip -qq Godot_v$linkVersion-stable_export_templates.tpz
	mv templates/* ~/.local/share/godot/export_templates/$version
	rm -r templates
	rm Godot_v$linkVersion-stable_export_templates.tpz
else
	echo "Export templates found..."
fi
```

### 2. Les presets d'exportation

Ensuite, j'ai suivi la documentation de Godot pour l'utiliser en <abbr title="Command Line Interface">CLI</abbr>. J'ai trouvé des commandes qui se sont avérées utiles pour réaliser ma mission.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-cli-doc.png?raw=true", alt="Godot CLI documentation", no_hover=true)}}

En observant les arguments, nous retrouvons <mark>preset</mark> et <mark>path</mark>. <strong>Path</strong> qui signifie tout simplement le chemin absolu ou relatif vers le projet godot, alors que <strong>Preset</strong> est un peu complexe à comprendre.

J'ai compris qu'ajouter un preset d'exportation au projet comme sur l'image suivante :

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-export-menu.png?raw=true", alt="Godot adding export option menu", no_hover=true) }}

Permet de générer, si non existant, le fichier `export_presets.cfg` à la racine du projet Godot et ajoute la configuration du preset dedans avec les options que nous lui affectons.

Voici à quoi ressemble un preset dans le fichier `export_presets.cfg` :

```txt
[preset.0]

name="Web"
platform="Web"
runnable=true
dedicated_server=false
custom_features=""
export_filter="all_resources"
include_filter="*.csv"
exclude_filter=""
export_path="Exported_'$name'/'$name'.html"
encryption_include_filters=""
encryption_exclude_filters=""
encrypt_pck=false
encrypt_directory=false

[preset.0.options]

custom_template/debug=""
custom_template/release=""
variant/extensions_support=false
vram_texture_compression/for_desktop=true
vram_texture_compression/for_mobile=false
html/export_icon=true
html/custom_html_shell=""
html/head_include=""
html/canvas_resize_policy=2
html/focus_canvas_on_start=true
html/experimental_virtual_keyboard=false
progressive_web_app/enabled=false
progressive_web_app/offline_page=""
progressive_web_app/display=1
progressive_web_app/orientation=0
progressive_web_app/icon_144x144=""
progressive_web_app/icon_180x180=""
progressive_web_app/icon_512x512=""
progressive_web_app/background_color=Color(0, 0, 0, 1)
```

### 3. Hébergé le projet exporté

Ensuite, pour ouvrir le projet exporté vers de l'HTML5, j'ai utilisé un serveur Python pour accéder aux fichiers dans mon navigateur. Voici le code que j'ai ajouté à mon script pour créer un fichier .py afin d'avoir un serveur Python pour héberger le projet exporté :

```bash
# Create a python virtual environment to install needed libs
echo "Check if python venv is installed..."
if ! dpkg -l | grep -q python3-venv;
then
	echo "Installing python venv..."
	sudo apt install python3-venv -y
fi

python3 -m venv Exported_$name/venv

# Installing libs in the python virtual env
echo "Creating python virtual environment..."
source Exported_$name/venv/bin/activate

echo "Installing python libs..."
python3 -m pip install Flask &>/dev/null
python3 -m pip install requests &>/dev/null


# Create a python server to host godot
echo "Creating server to host godot..."

godotServer="from http.server import SimpleHTTPRequestHandler
from http.server import HTTPServer
from socketserver import ThreadingMixIn
import os

class CustomRequestHandler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
        self.send_header('Access-Control-Allow-Headers', 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token')
        self.send_header('Cross-Origin-Embedder-Policy', 'require-corp')
        self.send_header('Cross-Origin-Opener-Policy', 'same-origin')
        super().end_headers()

    def translate_path(self, path):
        if path == '/':
            return os.path.join(os.getcwd(), '$name.html')
        return super().translate_path(path)

class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    pass

def run(server_class=ThreadedHTTPServer, handler_class=CustomRequestHandler, port=8000):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f'Starting server on port {port}...')
    httpd.serve_forever()

if __name__ == '__main__':
    run()"

echo -e "$godotServer" > Exported_$name/godotServer.py
```

Le début du code sert surtout à éviter d'installer les librairies nécessaires pour créer un serveur avec Python sur la machine hôte, mais plutôt sur un environnement virtuel.

### 4. Accèder aux ressources externes

Il y a une subtilité dans la tâche que j'ai dû réaliser : je devais créer un script pour automatiser l'exportation de projet Godot vers le web. Cependant, le projet sur lequel je devais tester mon script nécessitait un fichier CSV externe au projet.

La solution que j'ai adoptée pour récupérer les informations du CSV a été de l'héberger sur le dépôt GitLab du script, et d'utiliser un autre serveur Python avec les autorisations <abbr title="Cross-Origin Resource Sharing">CORS</abbr> nécessaires pour y accéder.

```bash
# Creating a server to handle request for godot project
echo "Creating python server to make request..."

requestServer="from flask import Flask, request, jsonify, send_from_directory
import requests

app = Flask(__name__)

@app.route('/proxy', methods=['GET'])
def proxy():
    # Récupérez l'URL de la demande client
    url = request.args.get('url')

    # Faites une requête HTTP vers Gitlab
    response = requests.get(url)

    # Renvoyez la réponse du serveur Gitlab avec les en-têtes CORS appropriés
    response_headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token',
        'Cross-Origin-Embedder-Policy': 'require-corp',
        'Cross-Origin-Opener-Policy': 'same-origin'
    }

    return (response.text, response.status_code, response_headers)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)"

echo -e "$requestServer" > Exported_$name/proxy_server.py
```

<details>
    <summary>Voir le script en entier</summary>

    #!/bin/bash

    scriptUsage () {
    	echo -e "Usage: $0 <godot_path> <export_preset>
    	Examples :
    		$0 godot Web
    		$0 /bin/godot Web
    		$0 Godot_v4.2.1.stable Web"
    }

    godot=$1

    if [[ $godot == "" ]];
    then
    	scriptUsage
    	exit 1
    fi


    preset=$2

    if [[ $preset == "" ]];
    then
    	scriptUsage
    	exit 2
    fi


    name=$(pwd | rev | cut -d'/' -f1 | rev)


    version=$($godot --version | cut -d "." -f 1,2,3,4 | sed 's/\.official//')

    if [[ $version == "" ]];
    then
    	echo "Wrong godot path!"
    	exit 5
    fi


    # Export preset to HTML5 (Web)
    echo "Setup export preset..."

    presetCfg='[preset.0]

    name="Web"
    platform="Web"
    runnable=true
    dedicated_server=false
    custom_features=""
    export_filter="all_resources"
    include_filter="*.csv"
    exclude_filter=""
    export_path="Exported_'$name'/'$name'.html"
    encryption_include_filters=""
    encryption_exclude_filters=""
    encrypt_pck=false
    encrypt_directory=false

    [preset.0.options]

    custom_template/debug=""
    custom_template/release=""
    variant/extensions_support=false
    vram_texture_compression/for_desktop=true
    vram_texture_compression/for_mobile=false
    html/export_icon=true
    html/custom_html_shell=""
    html/head_include=""
    html/canvas_resize_policy=2
    html/focus_canvas_on_start=true
    html/experimental_virtual_keyboard=false
    progressive_web_app/enabled=false
    progressive_web_app/offline_page=""
    progressive_web_app/display=1
    progressive_web_app/orientation=0
    progressive_web_app/icon_144x144=""
    progressive_web_app/icon_180x180=""
    progressive_web_app/icon_512x512=""
    progressive_web_app/background_color=Color(0, 0, 0, 1)'

    echo -e "$presetCfg" > export_presets.cfg


    # Check if there are already export templates
    echo "Check if there are export templates..."

    mkdir -p ~/.local/share/godot/export_templates/$version

    exportTpl=$(ls ~/.local/share/godot/export_templates/$version)

    linkVersion=$(echo $version | grep -oP '(\d+(\.\d+)+)')


    # Download export templates if there aren't already downloaded
    if [[ $exportTpl == "" ]];
    then
    	echo "Export templates not found..."
    	echo "Downloading export templates..."
    	wget -q https://github.com/godotengine/godot/releases/download/$linkVersion-stable/Godot_v$linkVersion-stable_export_templates.tpz
    	unzip -qq Godot_v$linkVersion-stable_export_templates.tpz
    	mv templates/* ~/.local/share/godot/export_templates/$version
    	rm -r templates
    	rm Godot_v$linkVersion-stable_export_templates.tpz
    else
    	echo "Export templates found..."
    fi

    mkdir -p Exported_$name

    echo "Exporting godot project..."
    $godot --headless --export-release $preset Exported_$name/$name.html &>/dev/null


    # Create a python virtual environment to install needed libs
    echo "Check if python venv is installed..."
    if ! dpkg -l | grep -q python3-venv;
    then
    	echo "Installing python venv..."
    	sudo apt install python3-venv -y
    fi

    python3 -m venv Exported_$name/venv

    # Installing libs in the python virtual env
    echo "Creating python virtual environment..."
    source Exported_$name/venv/bin/activate

    echo "Installing python libs..."
    python3 -m pip install Flask &>/dev/null
    python3 -m pip install requests &>/dev/null


    # Create a python server to host godot
    echo "Creating server to host godot..."

    godotServer="from http.server import SimpleHTTPRequestHandler
    from http.server import HTTPServer
    from socketserver import ThreadingMixIn
    import os

    class CustomRequestHandler(SimpleHTTPRequestHandler):
        def end_headers(self):
            self.send_header('Access-Control-Allow-Origin', '*')
            self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
            self.send_header('Access-Control-Allow-Headers', 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token')
            self.send_header('Cross-Origin-Embedder-Policy', 'require-corp')
            self.send_header('Cross-Origin-Opener-Policy', 'same-origin')
            super().end_headers()

        def translate_path(self, path):
            if path == '/':
                return os.path.join(os.getcwd(), '$name.html')
            return super().translate_path(path)

    class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
        pass

    def run(server_class=ThreadedHTTPServer, handler_class=CustomRequestHandler, port=8000):
        server_address = ('', port)
        httpd = server_class(server_address, handler_class)
        print(f'Starting server on port {port}...')
        httpd.serve_forever()

    if __name__ == '__main__':
        run()"

    echo -e "$godotServer" > Exported_$name/godotServer.py


    # Creating a server to handle request for godot project
    echo "Creating python server to make request..."

    requestServer="from flask import Flask, request, jsonify, send_from_directory
    import requests

    app = Flask(__name__)

    @app.route('/proxy', methods=['GET'])
    def proxy():
        # Récupérez l'URL de la demande client
        url = request.args.get('url')

        # Faites une requête HTTP vers Gitlab
        response = requests.get(url)

        # Renvoyez la réponse du serveur Gitlab avec les en-têtes CORS appropriés
        response_headers = {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
            'Access-Control-Allow-Headers': 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token',
            'Cross-Origin-Embedder-Policy': 'require-corp',
            'Cross-Origin-Opener-Policy': 'same-origin'
        }

        return (response.text, response.status_code, response_headers)

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8080)"

    echo -e "$requestServer" > Exported_$name/proxy_server.py

    exit 0

</details>

<section class="task-nav">
    <a href="/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">Toutes les tâches</span>
        </div>
    </a>
    <a href="/stage/graphique-3d-formations">
        <div>
            <span class="label">Deuxième tâche</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
