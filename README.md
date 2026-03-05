# Laravel JWT API (Docker + CI/CD + AWS ECS)

Backend API con Laravel 12 y autenticacion JWT (login/logout/me/refresh), contenedores Docker, pruebas automaticas y despliegue continuo a AWS ECS Fargate.

## 1. Requisitos

- PHP 8.3+
- Composer 2+
- Docker + Docker Compose
- AWS CLI v2
- Cuenta AWS con permisos para ECR, ECS, IAM, EC2 y (opcional) RDS

## 2. Variables de entorno

Crear `.env` desde el ejemplo:

```bash
cp .env.example .env
php artisan key:generate
php artisan jwt:secret
```

## 3. Endpoints JWT

- `POST /api/auth/login`
- `POST /api/auth/logout` (Bearer token)
- `GET /api/auth/me` (Bearer token)
- `POST /api/auth/refresh` (Bearer token)

Payload login:

```json
{
  "email": "user@example.com",
  "password": "secret123"
}
```

## 4. Ejecutar con Docker local

Levantar contenedores:

```bash
docker compose up -d --build
```

Migrar base de datos:

```bash
docker compose exec app php artisan migrate
```

Crear usuario de prueba:

```bash
docker compose exec app php artisan tinker
```

Dentro de tinker:

```php
\App\Models\User::create([
  'name' => 'Admin',
  'email' => 'admin@example.com',
  'password' => 'secret123'
]);
```

API local: `http://localhost:8080`

## 5. Ejecutar tests automaticos

```bash
php artisan test
```

## 6. CI con GitHub Actions

Archivo: `.github/workflows/ci.yml`

Se ejecuta en cada push/PR a `main`:

1. Instala PHP y Composer
2. Instala dependencias
3. Genera `APP_KEY` y `JWT_SECRET`
4. Ejecuta `php artisan test`

## 7. CD con GitHub Actions a AWS ECS

Archivo: `.github/workflows/cd.yml`

Flujo:

1. Build imagen Docker
2. Push a Amazon ECR
3. Render task definition
4. Deploy a ECS Fargate

### Secrets requeridos en GitHub

- `AWS_ROLE_ARN`
- `ECS_CLUSTER`
- `ECS_SERVICE`

## 8. Comandos paso a paso para Docker en AWS

### 8.1 Configurar AWS CLI

```bash
aws configure
```

### 8.2 Variables

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=123456789012
export ECR_REPOSITORY=laravel-jwt-api
export ECS_CLUSTER=laravel-jwt-cluster
export ECS_SERVICE=laravel-jwt-service
```

### 8.3 Crear repositorio ECR

```bash
aws ecr create-repository \
  --repository-name $ECR_REPOSITORY \
  --region $AWS_REGION
```

### 8.4 Login Docker en ECR

```bash
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

### 8.5 Build, tag y push de imagen

```bash
docker build -t $ECR_REPOSITORY .
docker tag $ECR_REPOSITORY:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest
```

### 8.6 Crear cluster ECS (Fargate)

```bash
aws ecs create-cluster --cluster-name $ECS_CLUSTER --region $AWS_REGION
```

### 8.7 Registrar task definition

Edita `.aws/task-definition.json` con tu `AWS_ACCOUNT_ID`, roles IAM y variables reales, luego:

```bash
aws ecs register-task-definition \
  --cli-input-json file://.aws/task-definition.json \
  --region $AWS_REGION
```

### 8.8 Crear servicio ECS

Necesitas subnets y security groups existentes.

```bash
aws ecs create-service \
  --cluster $ECS_CLUSTER \
  --service-name $ECS_SERVICE \
  --task-definition laravel-jwt-api-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --region $AWS_REGION
```

### 8.9 Forzar nuevo despliegue

```bash
aws ecs update-service \
  --cluster $ECS_CLUSTER \
  --service $ECS_SERVICE \
  --force-new-deployment \
  --region $AWS_REGION
```

## 9. Archivos importantes

- `app/Http/Controllers/Api/AuthController.php`
- `routes/api.php`
- `tests/Feature/AuthTest.php`
- `Dockerfile`
- `docker-compose.yml`
- `.github/workflows/ci.yml`
- `.github/workflows/cd.yml`
- `.aws/task-definition.json`

## 10. Notas de produccion

- Usa RDS MySQL en lugar del contenedor `db`.
- Nunca dejes `DB_PASSWORD` plano en task definition; usa AWS Secrets Manager.
- Agrega ALB + HTTPS (ACM) para trafico publico.
- Define `APP_KEY` y `JWT_SECRET` como secretos de entorno en ECS.

## 11. Trigger

- Commit de prueba para disparar nuevamente el workflow de CD.
- Commit de prueba adicional para validar disparo de CD (2026-03-05).
- Commit de prueba adicional para relanzar CD tras ajuste de permisos IAM.
- Commit de prueba adicional para validar CD con RDS configurado.
- Commit de prueba adicional para relanzar CI/CD con APP_KEY y JWT_SECRET configurados.
