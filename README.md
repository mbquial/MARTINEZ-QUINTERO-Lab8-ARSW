# Lab #8 — Infraestructura como Código con Terraform (Azure)

**Curso:** Arquitectura de Software - ARSW  
**Equipo:** MARTINEZ-QUINTERO  

---

## Integrantes del equipo

| Nombre | Perfil |
|--------|-----|
| María Quintero | [@mbquial](https://github.com/mbquial) |
| Nikolas Martinez | [@NikoMAR3](https://github.com/NikoMAR3) |

---

## Arquitectura desplegada

- **Resource Group:** `lab8-rg` (región: `eastus2`)
- **Virtual Network:** `lab8-vnet` (`10.10.0.0/16`) con subnets `subnet-web` y `subnet-mgmt`
- **Load Balancer Standard (L4):** `lab8-lb` con frontend IP pública estática
- **2x VM Ubuntu 22.04 LTS** (`Standard_D2as_v7`) con nginx instalado via cloud-init
- **Network Security Group:** permite HTTP (`80/TCP`) desde Internet y SSH (`22/TCP`) desde IP del equipo
- **Backend remoto:** Azure Storage Account `sttfstate6864` con state locking habilitado

---

## Estructura del repositorio

```
.
├── .github/
│   └── workflows/
│       └── terraform.yml        # Pipeline CI/CD con OIDC
├── infra/
│   ├── main.tf                  # Wiring de módulos y Resource Group
│   ├── providers.tf             # Provider azurerm + backend remoto
│   ├── variables.tf             # Variables de entrada
│   ├── outputs.tf               # Outputs (lb_public_ip, vm_names, rg)
│   ├── cloud-init.yaml          # Instalación automática de nginx
│   ├── backend.hcl.example      # Ejemplo de configuración del backend
│   └── env/
│       └── dev.tfvars           # Variables del entorno de desarrollo
├── modules/
│   ├── vnet/                    # Módulo: Virtual Network y subnets
│   ├── compute/                 # Módulo: NICs y VMs Linux
│   └── lb/                      # Módulo: Load Balancer, NSG y reglas
└── docs/
    ├── Reflexion.md/            # Reflexión pedida
    └── INSTALL_TERRAFORM.md     # Guía de instalación de Terraform
```
---
## Diagramas

![Componentes](/docs/img/componentes.png)

![Secuencia](/docs/img/diagramaSecuencia.png)

---

## Capturas de funcionamiento

![Funcionamiento de la VM0](/docs/img/vm0.png)

![Funcionamiento de la VM1](/docs/img/vm1.png)

![Máquinas pool](/docs/img/pool.png)

![Grupo de Recursos](/docs/img/grupos.png)

![gtactions](/docs/img/gtact.png)

![comentario](/docs/img/image.png)

---


## Cómo desplegar

### Requisitos previos

- Azure CLI instalado y autenticado (`az login`)
- Terraform >= 1.6 instalado
- SSH key generada (`ssh-keygen -t ed25519`)
- Storage Account para el backend creado (ver sección Bootstrap)

### Bootstrap del backend remoto

```bash
SUFFIX=$RANDOM
RG=rg-tfstate-lab8
STO=sttfstate${SUFFIX}

az group create -n $RG -l eastus2
az storage account create -g $RG -n $STO -l eastus2 --sku Standard_LRS --encryption-services blob
az storage container create --name tfstate --account-name $STO
```

### Despliegue local

```bash
# 1. Completar backend.hcl con los datos del Storage Account
cp infra/backend.hcl.example infra/backend.hcl
# (editar con los valores reales)

# 2. Inicializar con backend remoto
cd infra
terraform init -backend-config=backend.hcl

# 3. Revisar y planificar
terraform fmt -recursive
terraform validate
terraform plan -var-file=env/dev.tfvars -out=plan.tfplan

# 4. Aplicar
terraform apply "plan.tfplan"

# 5. Verificar
curl http://$(terraform output -raw lb_public_ip)
```

### Pipeline CI/CD

- Cada Pull Request hacia `main` ejecuta `terraform fmt`, `validate` y `plan` automáticamente
- El plan se publica como comentario en el PR para revisión del equipo
- El apply se ejecuta manualmente desde **Actions → Terraform CI/CD → Run workflow → apply**
- La autenticación con Azure usa **OIDC** (sin secretos de larga duración)

---

## Outputs del despliegue

| Output | Descripción |
|--------|-------------|
| `lb_public_ip` | IP pública del Load Balancer |
| `resource_group_name` | Nombre del Resource Group creado |
| `vm_names` | Lista de nombres de las VMs (`["lab8-vm-0", "lab8-vm-1"]`) |

---

## Destrucción de recursos

```bash
terraform destroy -var-file=env/dev.tfvars
```

Verificar en el portal de Azure que el Resource Group `lab8-rg` haya sido eliminado. El RG del tfstate (`rg-tfstate-lab8`) puede conservarse o eliminarse con:

```bash
az group delete --name rg-tfstate-lab8 --yes
```

---

## Preguntas de reflexión

**¿Por qué L4 LB vs Application Gateway (L7)?**  
El L4 LB es suficiente para distribuir tráfico HTTP básico entre VMs homogéneas. El Application Gateway es mejor cuando es necesario routing por URL, terminación TLS o WAF. Nada de lo anterior se tiene que aplicar para este laboratorio.

**¿Qué implicaciones de seguridad tiene exponer 22/TCP?**  
Aunque está restringido al CIDR del estudiante, el puerto SSH sigue siendo un vector de ataque si la IP cambia o se comparte. En producción se reemplazaría por Azure Bastion.
