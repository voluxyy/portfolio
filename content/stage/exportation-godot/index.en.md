+++
title = "Godot Exportation"
+++

## 1. Automating Godot Project Export

The first task I had during my internship was to automate the project export process for the Godot engine.

### 1. Export Templates

I started by using Godot and exporting a project from the software's graphical interface.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-export-menu.png?raw=true", alt="Godot export menu", no_hover=true)}}

Using the interface allowed me to understand and identify the different important settings for exporting a project.

Notably, during the first export, if the software does not detect an export template, it offers to download one from the mirror of our choice.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-template-download1.png?raw=true", alt="Godot template download", no_hover=true)}}

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-template-download2.png?raw=true", alt="Godot template download", no_hover=true)}}

Following this discovery, I added the following code to my script to check if a template of the correct version already exists. If not, I download it from GitHub.

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

### 2. Export Presets

Next, I followed the Godot documentation to use it in <abbr title="Command Line Interface">CLI</abbr>. I found commands that proved useful for completing my task.

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-cli-doc.png?raw=true", alt="Godot CLI documentation", no_hover=true)}}

When looking at the arguments, we find <mark>preset</mark> and <mark>path</mark>. **Path** simply means the absolute or relative path to the Godot project, while **Preset** is a bit more complex to understand.

I understood that adding an export preset to the project, as shown in the following image:

{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/godot-export-menu.png?raw=true", alt="Godot adding export option menu", no_hover=true) }}

Allows generating, if not already existing, the `export_presets.cfg` file at the root of the Godot project and adds the preset configuration inside it with the options we assign to it.

Here’s what a preset looks like in the `export_presets.cfg` file:

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

### 3. Hosting the Exported Project

Then, to open the exported project in HTML5, I used a Python server to access the files in my browser. Here is the code I added to my script to create a .py file to have a Python server to host the exported project:

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

The beginning of the code mainly serves to avoid installing the necessary libraries for creating a Python server on the host machine but instead in a virtual environment.

### 4. Accessing External Resources

There was a nuance in the task I had to complete: I needed to create a script to automate the export of a Godot project to the web. However, the project I was testing my script on required an external CSV file that was not part of the project.

The solution I adopted to retrieve the information from the CSV was to host it on the GitLab repository of the script and, using another Python server with the necessary <abbr title="Cross-Origin Resource Sharing">CORS</abbr> permissions, access it.

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
    <summary>View full script</summary>

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
    <a href="/portfolio/en/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">All tasks</span>
        </div>
    </a>
    <a href="/portfolio/en/stage/graphique-3d-formations">
        <div>
            <span class="label">Second task</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
