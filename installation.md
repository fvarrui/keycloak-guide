# Installation

1. [Download](https://www.keycloak.org/downloads) KC server (bare metal edition).
   
   ```bash
   wget https://github.com/keycloak/keycloak/releases/download/x.y.z/keycloak-x.y.z.zip
   ```

2. Unzip the downloaded file:
   
   ```bash
   unzip keycloak-x.y.z.zip
   ```

3. Run the server in **production** mode:
   
   ```bash
   cd keycloak-x.y.z/bin
   ./kc start
   ```
   
   Or in **development** mode:
   
   ```bash
   cd keycloak-x.y.z/bin
   ./kc start-dev
   ```

4. Connect to KC using a browser:
   
   ```
   http://localhost:8080
   https://localhost:8443
   ```
