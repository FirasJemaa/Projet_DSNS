wp_theme_path: "/var/www/wordpress/wordpress/wp-content/themes/itway-theme"

- name: Créer le dossier du thème WordPress ITWay
    file:
    path: "{{ wp_theme_path }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'

- name: Copier le fichier style.css
    template:
        src: "style.css"
        dest: "{{ wp_theme_path }}/style.css"


- name: Copier index.php
    template:
        src: "index.php"
        dest: "{{ wp_theme_path }}/index.php"

- name: Copier main.js
    template:
        src: "main.js"
        dest: "{{ wp_theme_path }}/main.js"

- name: Activer le thème avec wp-cli
    shell: wp theme activate itway-theme
    args:
    chdir: /var/www/html
    become_user: www-data
    ignore_errors: true
