# Reflexión Técnica — Lab #8

**Curso:** Arquitectura de Software - ARSW  
**Equipo:** MARTINEZ-QUINTERO  

---

## 1. Decisiones de diseño

La arquitectura se estructuró en tres módulos Terraform independientes (`vnet`, `compute`, `lb`), siguiendo el principio de separación de responsabilidades. Esta decisión permite reutilizar cada módulo en otros entornos o proyectos sin cambiar el resto de la infraestructura.

Se usó el Load Balancer de nivel L4 (TCP) de Azure en lugar del Application Gateway (L7) porque el requisito del laboratorio es únicamente distribuir tráfico HTTP básico entre dos VMs que sirven páginas estáticas. El L4 LB es más sencillo de configurar, tiene menor costo y menor latencia al no necesitar inspección de la capa de aplicación.

---

## 2. Trade-offs identificados

### L4 Load Balancer vs Application Gateway (L7)

| Aspecto | L4 Load Balancer | Application Gateway (L7) |
|---------|-----------------|--------------------------|
| Costo | ~$0.005/hora | ~$0.25/hora |
| Complejidad | Baja | Media-alta |
| Routing por URL | No | Sí |
| Terminación TLS | No | Sí |
| WAF integrado | No | Sí |
| **Decisión** | ✅ Elegido para este lab | Para producción con múltiples microservicios |

### SSH expuesto vs Azure Bastion

El puerto `22/TCP` está restringido al CIDR del estudiante mediante la variable `allow_ssh_from_cidr`, lo que reduce el riesgo de acceso no autorizado. Sin embargo, en un entorno productivo la práctica recomendada es usar **Azure Bastion**, que elimina la necesidad de exponer SSH directamente a Internet.

El trade-off es económico: Bastion tiene un costo adicional de ~$0.19/hora, lo que lo hace inviable para este laboratorio.

### VM size: Standard_D2as_v7 vs Standard_B1s

`Standard_B1s` (1 vCPU, 1 GB RAM) era el tamaño ideal por costo, pero no estaba disponible en las regiones `eastus` ni `eastus2` al momento del despliegue debido a restricciones de capacidad en cuentas de estudiante. Se optó por `Standard_D2as_v7` (2 vCPU, 8 GB RAM), que tiene mayor capacidad de la necesaria pero garantizó disponibilidad inmediata.

---

## 3. Estimación de costos

| Recurso | Costo/hora | Costo estimado (4h) |
|---------|-----------|---------------------|
| 2x VM Standard_D2as_v7 | ~$0.192 | ~$0.77 |
| Azure Load Balancer Standard | ~$0.025 | ~$0.10 |
| Public IP (estática) | ~$0.004 | ~$0.02 |
| Storage Account (tfstate) | ~$0.002 | ~$0.01 |
| VNet / Subnets | $0.00 | $0.00 |
| **Total estimado** | **~$0.223/h** | **~$0.90 por sesión** |

---

## 4. Destrucción segura de recursos

Al finalizar el laboratorio, todos los recursos deben eliminarse para evitar cargos no deseados. El proceso recomendado es:

1. Ejecutar `terraform destroy -var-file=env/dev.tfvars` y confirmar con `yes`.
2. Verificar en el portal de Azure que el Resource Group `lab8-rg` haya sido eliminado.
3. El Resource Group `rg-tfstate-lab8` (backend del state) puede conservarse si se planea reutilizar el backend, o eliminarse con:

```bash
az group delete --name rg-tfstate-lab8 --yes
```

4. Etiquetar siempre los recursos con `expires` para facilitar la auditoría de costos y la limpieza automática.

---

## 5. Mejoras para un entorno productivo

- **Autoscaling:** migrar a VM Scale Sets con políticas de escalado automático según CPU o tráfico de red.
- **Observabilidad:** integrar Azure Monitor con alertas sobre el estado del health probe y métricas de las VMs.
- **Seguridad:** reemplazar SSH directo por Azure Bastion; usar Key Vault para gestionar secretos y certificados.
- **Alta disponibilidad:** distribuir las VMs en Availability Zones distintas para tolerancia a fallos de zona.
- **CI/CD:** agregar un entorno de `staging` con aprobación manual antes de promover cambios a producción.
- **TLS:** agregar un Application Gateway con certificado SSL/TLS para cifrar el tráfico en tránsito.
