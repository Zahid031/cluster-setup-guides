# Kafbat UI with RBAC Setup Guide

This guide will walk you through setting up Kafbat UI with Role-Based Access Control (RBAC) using Keycloak for authentication and authorization.

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of OAuth2 and RBAC concepts
- Access to a Kafka cluster (replace connection details in configuration)

## Project Structure

Create the following directory structure:

```
project-root/
├── docker-compose.yml
├── app-config.yml
├── roles.yml
└── keycloak/
    ├── data/          (will be created automatically)
    └── import/        (optional: for realm JSON import)
```

## Step 1: Create Docker Compose Configuration

Create `docker-compose.yml` with the following content:

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.3.4
    container_name: keycloak
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    command: start-dev        # if want to import realm then "start-dev --import-realm"
    volumes:
      - ./keycloak/data:/opt/keycloak/data      # persistent data: users, realms, roles
      - ./keycloak/import:/opt/keycloak/data/import  # import realm folder
    ports:
      - "8080:8080"
    networks:
      - kafkaUI-net

  kafbat-ui:
    image: ghcr.io/kafbat/kafka-ui:main
    container_name: kafbat-ui
    environment:
      SPRING_CONFIG_ADDITIONAL-LOCATION: /app-config.yml,/roles.yml
    volumes:
      - ./app-config.yml:/app-config.yml:ro
      - ./roles.yml:/roles.yml:ro
    ports:
      - "9000:8080"
    depends_on:
      - keycloak
    networks:
      - kafkaUI-net

networks:
  kafkaUI-net:
    driver: bridge
```

## Step 2: Configure Kafbat UI OAuth2 Client

Create `app-config.yml` with the following configuration:

```yaml
kafka:
  clusters:
    - name: local
      bootstrapServers: "PLAINTEXT://your-broker-host:9092"  # Replace with your Kafka broker

auth:
  type: OAUTH2
  oauth2:
    client:
      keycloak:
        clientId: kafbat-client
        clientSecret: kafbat-secret     # Will be generated in Keycloak
        scope: openid
        client-name: keycloak
        provider: keycloak
        # IMPORTANT: Update with your actual host/port
        redirect-uri: http://<host-ip>:9000/login/oauth2/code/keycloak
        authorization-grant-type: authorization_code
        # Replace 'kafbat' with your realm name
        issuer-uri: http://<keycloak-server-ip>:8080/realms/kafbat
        jwk-set-uri: http://<keycloak-server-ip>:8080/realms/kafbat/protocol/openid-connect/certs
        user-name-attribute: preferred_username
        custom-params:
          type: oauth
          # Field name in JWT token that contains role names
          roles-field: roles
```

**Important Configuration Notes:**
- Replace `your-broker-host:9092` with your actual Kafka broker connection
- Update `redirect-uri` if accessing from a different host/port
- The `clientSecret` will be generated in Keycloak (Step 4)

## Step 3: Define RBAC Roles and Permissions

Create `roles.yml` to define role-based access control:

```yaml
rbac:
  roles:
    - name: "admins"
      clusters:
        - local
      subjects:
        - provider: oauth
          type: role
          value: "kafbat-admin"   # Keycloak role name
      permissions:
        - resource: applicationconfig
          actions: all
        - resource: clusterconfig
          actions: all
        - resource: topic
          value: ".*"
          actions: all
        - resource: consumer
          value: ".*"
          actions: all
        - resource: schema
          value: ".*"
          actions: all
        - resource: connect
          value: ".*"
          actions: all
        - resource: ksql
          actions: all
        - resource: acl
          actions: [ view ]

    - name: "readonly"
      clusters:
        - local
      subjects:
        - provider: oauth
          type: role
          value: "kafbat-readonly"   # Keycloak role name
      permissions:
        - resource: clusterconfig
          actions: [ "view" ]
        - resource: topic
          value: ".*"
          actions:
            - VIEW
            - MESSAGES_READ
            - ANALYSIS_VIEW
        - resource: consumer
          value: ".*"
          actions: [ view ]
        - resource: schema
          value: ".*"
          actions: [ view ]
        - resource: connect
          value: ".*"
          actions: [ view ]
        - resource: acl
          actions: [ view ]
```

**Role Definitions:**
- **admins**: Full access to all Kafbat UI features
- **readonly**: View-only access to topics, consumers, schemas, etc.

## Step 4: Start Services

1. Start the services:
```bash
docker-compose up -d
```

2. Wait for services to start (check logs if needed):
```bash
docker-compose logs -f
```

## Step 5: Configure Keycloak

### Access Keycloak Admin Console

1. Open your browser and navigate to: `http://<keycloak-server>:8080`
2. Login with:
   - **Username**: `admin`
   - **Password**: `admin`

### Create Realm

1. Click on the dropdown next to "Master" realm (top-left)
2. Click "Create Realm"
3. Set **Realm name**: `kafbat`
4. Click "Create"

### Create Realm Roles

1. In the `kafbat` realm, go to **Realm roles**
2. Click "Create role"
3. Create two roles:
   - **Role name**: `kafbat-admin`
   - **Role name**: `kafbat-readonly`

### Create OAuth2 Client

1. Go to **Clients** → "Create client"
2. **General Settings**:
   - **Client type**: `OpenID Connect`
   - **Client ID**: `kafbat-client`
   - Click "Next"
3. **Capability config**:
   - **Client authentication**: `On` (confidential)
   - **Authorization**: `Off`
   - **Standard flow**: `On`
   - Click "Next"
4. **Login settings**:
   - **Valid redirect URIs**: `http://<server-ip>:9000/*`
   - **Valid post logout redirect URIs**: `http://<server-ip>:9000/*`
   - Click "Save"

### Configure Client Secret

1. In the created client, go to **Credentials** tab
2. Copy the **Client secret**
3. Update `app-config.yml`:
   ```yaml
   clientSecret: your-copied-secret-here
   ```

### Configure Role Mapping in JWT

1. In the client, go to **Client scopes** tab
2. Click on `kafbat-client-dedicated`
3. Go to **Mappers** tab → "Add mapper" → "By configuration"
4. Select **User Realm Role**
5. Configure:
   - **Name**: `realm-roles`
   - **Token Claim Name**: `roles`
   - **Add to ID token**: `On`
   - **Add to access token**: `On`
   - Click "Save"

### Create Users

1. Go to **Users** → "Create new user"
2. Create admin user:
   - **Username**: `test-admin`
   - **First name**: `test`
   - Click "Create"
   - Go to **Credentials** tab → "Set password"
   - Set password and uncheck "Temporary"
   - Go to **Role mapping** tab → "Assign role"
   - Select `kafbat-admin` role

3. Create readonly user:
   - **Username**: `test-user`
   - **First name**: `test-user`
   - **Last name**: `Readonly`
   - Click "Create"
   - Go to **Credentials** tab → "Set password"
   - Set password and uncheck "Temporary"
   - Go to **Role mapping** tab → "Assign role"
   - Select `kafbat-readonly` role

## Step 6: Restart Kafbat UI

After updating the client secret in `app-config.yml`:

```bash
docker-compose restart kafbat-ui
```

## Step 7: Test the Setup

1. Open Kafbat UI: `http://<server-ip>:9000`
2. You should be redirected to Keycloak login
3. Test with both users:
   - **alice** (admin): Should have full access
   - **bob** (readonly): Should have view-only access

## Troubleshooting

### Common Issues

1. **Authentication Loop**: Check redirect URIs match exactly
2. **No Roles in Token**: Verify role mapper is configured correctly
3. **Access Denied**: Check role names match between Keycloak and `roles.yml`
4. **Connection Issues**: Verify network connectivity between containers

### Useful Commands

```bash
# View logs
docker-compose logs keycloak
docker-compose logs kafbat-ui

# Restart services
docker-compose restart

# Stop all services
docker-compose down

# Clean up volumes (removes all data)
docker-compose down -v
```

## Optional: Realm Import/Export

### Export Realm Configuration

1. In Keycloak admin, go to **Realm settings** → **Action** → "Export"
2. Save the JSON file to `./keycloak/import/kafbat-realm.json`
3. Next startup will automatically import this configuration

### Import Existing Realm

1. Place realm JSON file in `./keycloak/import/`
2. Ensure docker-compose includes `--import-realm` flag
3. Restart Keycloak service

## Security Considerations

- Change default Keycloak admin credentials in production
- Use strong client secrets
- Configure proper SSL/TLS certificates
- Restrict redirect URIs to specific domains
- Regularly review and audit user roles and permissions

## Conclusion

You now have Kafbat UI running with OAuth2 authentication via Keycloak and role-based access control. Admin users can perform all operations while readonly users have limited view-only access to your Kafka clusters.
