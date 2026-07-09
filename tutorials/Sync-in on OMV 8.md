# Sync-in Server on openmediavault 8

[**Openmediavault**](https://www.openmediavault.org) (OMV) is an open-source network attached storage (NAS) solution based on Debian Linux.  
[**Sync-in**](https://sync-in.com) is an open-source platform to store, share, collaborate, and sync files.  
Among many other features, Sync-in includes WebDAV integration, as well as collaborative documents editing through integrations with [Collabora Online](https://www.collaboraonline.com) and [OnlyOffice](https://www.onlyoffice.com). Openmediavault offers a wide range of network storage related services, as well as Kubernetes and Compose plugins for installing containerized apps.

This guide describes the steps towards installing Sync-in Server and Collabora Online as **separate** docker containers on an openmediavault 8 system, using its [Docker Compose plugin](https://wiki.omv-extras.org/doku.php?id=omv8:omv8_plugins:docker_compose) as well as [Nginx Proxy Manager](https://nginxproxymanager.com) (NPM) to expose the private network Web services to the internet. 

### Background

The Sync-in Server [installation guide](https://sync-in.com/docs/category/install--operate) describes an elaborate hands-on process of downloading the latest release, issuing curl commands, modifying the configuration file 'envrionment.yaml', modifying the file 'compose.yaml' and issuing commands, all of which not being my cuppa tea.  
As no Sync-in compose file is offered, I set towards attempting to integrate this process into openmediavault Compose. I had hopes of incorporating the 'environment.yaml' into compose environment variables, but alas that will not do, as Sync-in components will require access to this file during the setup process. As a result, this guide will involve some manual steps, as well as editing the Sync-in configuration file 'environment.yaml'.

## Prerequisites

- An openmediavault 8 NAS set up and running
  - With Docker Compose plugin installed
  - With Nginx Proxy Manager docker container installed and running
- Remote access to the NAS using WinSCP

## Outline of steps

1. Installation of Sync-in docker using Compose
2. Proxy Host configuration for Sync-in using NPM
3. Installation of Collabora Online docker using Compose
4. Proxy Host configuration for Collabora Online using NPM

---

### 1. Installation of Sync-in docker using Compose

- Download the latest 'sync-in-compose.zip' from [**Sync-in server releases**](https://github.com/Sync-in/server/releases/) to your PC
- Also download the full Sync-in '[**environment.dist.yaml**](https://github.com/Sync-in/server/blob/main/environment/environment.dist.yaml)' configuration template
- Open your openmediavault workbench - Services - Compose - Files
- Create a new Compose file called 'sync-in-docker'
- Copy the contents of the 'docker-compose.yaml' from the zip file into the OMV Compose File editor
- Save the newly created compose file for now
- Compose will now have created a folder '/appdata/sync-in-docker' with some files
- Use WinSCP to remote access this folder '/appdata/sync-in-docker'
- Copy the contents of the downloaded 'sync-in-compose.zip' in there, which will result in the following structure:

```
/appdata/sync-in-docker/
├── config
│ ├── collabora
│ │ └── docker-compose.collabora.yaml
│ ├── nginx
│ │ ├── collabora.conf
│ │ ├── docker-compose.nginx.yaml
│ │ ├── nginx.conf
│ │ └── onlyoffice.conf
│ ├── onlyoffice
│ │ └── docker-compose.onlyoffice.yaml
│ └── sync-in-desktop-releases
│ ├── docker-compose.sync-in-desktop-releases.yaml
│ └── update.sh
├── docker-compose.yaml
└── environment.yaml
```

- (Much of this will not be used, but does offer a nice reference)
- Skipping the default minimal setup and going full whammy right away;  
open the file 'environment.yaml' and replace its full contents with that of the downloaded 'environment.dist.yaml'
- Search for the following items and edit these ALL_CAPS values with your own (random generated) ones:

```
...
mysql:
  url: 'mysql://root:ALPHANUMERIC_DB_KEY@mariadb:3306/sync_in'
...
auth:
  encryptionKey: 'ALPHANUMERIC_ENCRYPTION_KEY'
  token:
    access:
      secret: 'ALPHANUMERIC_ACCESS_KEY'
    refresh:
      secret: 'ALPHANUMERIC_REFRESH_KEY'
...
applications:
  files:
    dataPath: /app/data     # Sync-in internal data path. Needs to be exposed to SMB share in Compose
...
    sampleDocuments: [opendocument, microsoft]
...
    collabora:
      enabled: true
      externalServer: https://collabora.YOUR-DOMAIN.TLD
...
```

- Return to the OMV workbench - Services - Compose - Files
- Edit the compose file 'sync-in-docker' as follows, changing the ALL_CAPS credentials for your own:

```
include:
#  - ./config/nginx/docker-compose.nginx.yaml             # Not installed, as we'll be using Nginx Proxy Manager (NPM)
#  - ./config/collabora/docker-compose.collabora.yaml     # To be installed separately, not included
#  - ./config/onlyoffice/docker-compose.onlyoffice.yaml   # To be installed separately, not included
#  Enable the next line when applications.appStore.repository in environment.yaml is set to 'local'
#  - ./config/sync-in-desktop-releases/docker-compose.sync-in-desktop-releases.yaml  

name: sync-in
services:
  sync_in:
    image: syncin/server:2
    container_name: sync-in
    restart: always
    environment:
      - PUID=${{ uid:"appuser" }}           # omv user 'appuser'
      - PGID=${{ gid:"users" }}             # omv usergroup 'users'
      - TZ=${{ tz }}                        # omv system time zone
      - INIT_ADMIN=true
      - INIT_ADMIN_LOGIN=SYNC-IN-ADMIN      # AVOID $, / or other special characters that may trigger undesired parsing
      - INIT_ADMIN_PASSWORD=SYNC-IN-PASS    # AVOID $, / or other special characters that may trigger undesired parsing
    ports:
      - 8080:8080                           # External port:Internal port as defined in environment.yaml
    volumes:
      - ./environment.yaml:/app/environment/environment.yaml  # IMPORTANT: environment.yaml is required for sync-in internals
      - ${{ sf:"data" }}/sync-in:/app/data  # omv share 'data'/sync-in:internal dataPath as defined in environment.yaml
                                            # Location for desktop releases if appStore is set to local in environment.yaml
      - ${{ sf:"data" }}/sync-in/desktop_releases:/app/static/releases:ro
    depends_on:
      - mariadb
    logging:
      driver: json-file
      options:
        max-size: 25m
        max-file: 5
    networks:
      - sync_in_network
      - web-proxy                           # Common docker bridge network re. Nginx Proxy Manager 

  mariadb:
    image: mariadb:11
    container_name: mariadb
    restart: always
    command: --innodb_ft_cache_size=16000000 --max-allowed-packet=1G
    environment:
      MYSQL_ROOT_PASSWORD: ALPHANUMERIC_DB_KEY    # Same ALPHANUMERIC_DB_KEY as in mysql.url in environment.yaml
      MYSQL_DATABASE: sync_in
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - sync_in_network

networks:
  sync_in_network:
    name: sync_in_network                   # IMPORTANT: Prevent automatic prefixing of the docker network name
    driver: bridge
  web-proxy:                                # Common docker bridge network re. Nginx Proxy Manager 
    external: true

volumes:
  data:
  mariadb_data:
  desktop_releases:
```

- Save the compose file 
- Compose Check for issues, Compose Pull and Compose UP

If all is well, Sync-in should be accessible through [http://YOUR-SERVER-IP:8080](http://YOUR-SERVER-IP:8080) using above ADMIN credentials.

---

### 2. Proxy Host configuration for Sync-in

- Open your Nginx Proxy Manager (NPM) instance  
Note: NPM and Sync-in share the same docker bridge network 'web-proxy', so we can reference Sync-in by its container name
- Create a new Proxy Host:
  - Details:
    - Domain name: sync-in.YOUR-DOMAIN.TLD
    - Scheme: http
    - Forward hostname/IP: sync-in
    - Forward port: 8080
    - Option: Block common exploits
    - Option: Websockets support
  - SSL:
    - SSL Certificate: sync-in.YOUR-DOMAIN.TLD
    - Option: Force SSL
    - Option: HTTP/2 Support
    - Option: HSTS enabled
- Save

If all is well, Sync-in will now be accessible as well through [https://sync-in.YOUR-DOMAIN.TLD](https://sync-in.YOUR-DOMAIN.TLD).

---

### 3. Installation of Collabora Online docker using Compose

- Open your openmediavault workbench - Services - Compose - Files
- Create a new Compose file called 'collabora' with the following contents:

```
name: collabora
services:
  code:
    container_name: collabora
    image: collabora/code
    restart: always
    environment:
      - PUID=${{ uid:"appuser" }}          # omv user 'appuser'
      - PGID=${{ gid:"users" }}            # omv usergroup 'users'
      - TZ=${{ tz }}                       # omv system time zone
      - user=${{ uid:"appuser" }}          # omv user 'appuser'
      - username=COLLABORA-ADMIN           # AVOID $, / or other special characters that may trigger undesired parsing
      - password=COLLABORA-PASSWORD        # AVOID $, / or other special characters that may trigger undesired parsing
      - dictionaries=de_DE en_GB en_US es_ES fr_FR it nl pt_BR pt_PT ru
                                           # extra_parms (adapted from sync-in collabora compose)
      - extra_params=--o:ssl.enable=false --o:ssl.termination=true --o:logging.disable_server_audit=true --o:admin_console.enable=true
      - domain=collabora.YOUR-DOMAIN.TLD
    ports:
      - 9980:9980                          # Standard port redirection
    tty: true                              # keep the container running, equivalent to -t command line option
    cap_drop:                              # capabilities drop (as per sync-in collabora compose)
      - ALL
    cap_add:                               # capabilities add (as per sync-in collabora compose)
      - SYS_CHROOT
      - SYS_ADMIN
      - FOWNER
      - CHOWN
    logging:
      driver: json-file
      options:
        max-size: 25m
        max-file: 5
#    volumes:                                # IMPORTANT: TO BE UNCOMMENTED LATER ON
#      - ./coolwsd/coolwsd.xml:/etc/coolwsd/coolwsd.xml:rw  # mount local copy of coolwsd.xml config file
    networks:
      - sync_in_network                    # Common docker bridge network re. Sync-in
      - web-proxy                          # Common docker bridge network re. Nginx Proxy Manager

networks:
  sync_in_network:
    external: true
  web-proxy:
    external: true
```

- Save the compose file
- Compose Check for issues, Compose Pull and Compose UP

The following steps are neccessary to make a copy of the docker internal Collabora configuration file into a location that is better accessible. For this the Collabora docker container needs to be UP.

- Use WinSCP to remote access OMV's /appdata/collabora
- Create a subfolder ‘coolwsd’
- Use the context menu option “Static Custom Commands” - “Enter” and paste the command:  
`docker cp collabora:/etc/coolwsd/coolwsd.xml ./coolwsd/coolwsd.xml`
- Check if the file `/appdata/collabora/coolwsd/coolwsd.xml` is in place
- Ambitious people can get their hands dirty with advanced configuration of Collabora Online. I didn't find anything appealing.
- With `/appdata/collabora/coolwsd/coolwsd.xml` in place, WinSCP can be closed

Return to your openmediavault workbench - Services - Compose - Files

- Docker Compose DOWN collabora
- Open the collabora compose file and remove the # comment marks in front of the following two lines:

```
   volumes:
     - ./coolwsd/coolwsd.xml:/etc/coolwsd/coolwsd.xml:rw  # mount local copy of coolwsd.xml config file
```

- Make sure that all indenting is consistent, as this is a requirement for .yaml files
- Save the compose file
- Compose Check for issues and Compose UP

---

### 4. Proxy Host configuration for Collabora

- Open your Nginx Proxy Manager (NPM) instance  
Note: NPM and Collabora share the same docker bridge network 'web-proxy', so we can reference collabora by its container name
- Create a new Proxy Host:
  - Details:
    - Domain name: collabora.YOUR-DOMAIN.TLD
    - Scheme: http
    - Forward hostname/IP: collabora
    - Forward port: 9980
    - Option: Block common exploits
    - Option: Websockets support
  - Custom Locations:
    - /browser/
      - Scheme: http
      - Forward hostname/IP: collabora
      - Forward port: 9980
    - /hosting/
      - Scheme: http
      - Forward hostname/IP: collabora
      - Forward port: 9980
    - /cool/
      - Scheme: http
      - Forward hostname/IP: collabora
      - Forward port: 9980
  - SSL:
    - SSL Certificate: collabora.YOUR-DOMAIN.TLD
    - Option: Force SSL
    - Option: HTTP/2 Support
    - Option: HSTS enabled
- Save
- In the NPM Proxy Hosts list you can click the newly created host 'collabora' and it will open with a mere 'OK', indicating its status.  
Collabora in itself is a quiet servant ;-)

If all is well, you can now create new OpenDocument or Microsoft Office files in Sync-in and open them in Collabora Online.

## Result

Sync-in and Collabora Online are now installed as separate docker containers in openmediavault 8 Compose and proxied and SSL-terminated by Nginx Proxy Manager.  
The two containers can be managed from Compose. Take note that Collabora is dependant upon sync_in_network, meaning that Collabora can only be UP-ed after Sync-in, and Sync-in cannot be DOWN-ed if Collabora is still up.  
Therefor observe the following UP-DOWN cycle:

1. UP Sync-in
2. UP Collabora
3. DOWN Collabora
4. DOWN Sync-in

### External bridge network 'sync_in_network'

If the Docker bridge network 'sync_in_network' is defined externally, rather than inside the sync-in-docker, then there is no  interdependency between Sync-in and Collabora. Each can join the pre-existing bridge network sync_in_network independently.

In order to achieve this:

- DOWN both containers
- Manually create the docker bridge network 'sync_in_network'
- Modify the compose file 'sync-in-docker' similarly to 'collabora'

```
networks:
  sync_in_network:
    external: true
  web-proxy:
    external: true
```

- UP both containers
