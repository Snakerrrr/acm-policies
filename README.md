<<<<<<< HEAD
acm-policies
=======
# ACM Policies - Políticas de Compliance para OpenShift

Este repositorio contiene políticas de **Advanced Cluster Management (ACM)** para gestionar el **Compliance Operator** de Red Hat OpenShift y ejecutar escaneos de compliance automatizados usando perfiles CIS y PCI-DSS.

## 📋 Tabla de Contenidos

- [Descripción](#descripción)
- [Características](#características)
- [Requisitos](#requisitos)
- [Estructura del Repositorio](#estructura-del-repositorio)
- [Componentes](#componentes)
- [Configuración](#configuración)
- [Instalación](#instalación)
- [Uso](#uso)
- [Políticas Incluidas](#políticas-incluidas)
- [Placement y Selectores](#placement-y-selectores)
- [Troubleshooting](#troubleshooting)

## 📖 Descripción

Este proyecto utiliza **Kustomize** y el **Policy Generator** de ACM para desplegar y gestionar políticas de compliance en clusters OpenShift. Las políticas automatizan:

- La instalación del Compliance Operator
- La configuración de escaneos periódicos
- La ejecución de escaneos de compliance con perfiles CIS y PCI-DSS

## ✨ Características

- ✅ **Instalación automatizada** del Compliance Operator
- ✅ **Escaneos periódicos** configurados con cron (diarios a la 1 AM)
- ✅ **Perfiles de compliance**:
  - **CIS**: Center for Internet Security Benchmark
  - **PCI-DSS**: Payment Card Industry Data Security Standard
- ✅ **Gestión centralizada** mediante ACM Policies
- ✅ **Placement selectivo** para aplicar políticas a clusters específicos
- ✅ **Almacenamiento configurado** con rotación automática de resultados

## 🔧 Requisitos

- **OpenShift Container Platform** 4.x o superior
- **Advanced Cluster Management (ACM)** instalado y configurado
- **Policy Generator** habilitado en el hub cluster
- **Kustomize** (incluido en `kubectl` desde la versión 1.14+)
- Acceso de administrador al hub cluster de ACM
- Clusters objetivo etiquetados apropiadamente (ver [Placement y Selectores](#placement-y-selectores))

## 📁 Estructura del Repositorio

```
acm-policies/
├── base/
│   ├── kustomization.yaml                    # Configuración de Kustomize
│   ├── policy-generator-config.yaml          # Configuración del Policy Generator
│   └── compliance-operator/
│       ├── install/
│       │   └── compliance-operator.yaml       # Instalación del operador
│       ├── scan-setting/
│       │   └── scan-setting.yaml             # Configuración de escaneos periódicos
│       └── profiles/
│           ├── cis/
│           │   └── scansettingbinding-cis.yaml    # Binding CIS
│           └── pci/
│               └── scansettingbinding-pci.yaml   # Binding PCI-DSS
└── README.md
```

## 🧩 Componentes

### 1. Policy Generator Config (`policy-generator-config.yaml`)

Define las políticas de ACM que se generarán. Incluye:

- **Política de instalación**: Instala el Compliance Operator
- **Política de scan-setting**: Configura escaneos periódicos
- **Política CIS**: Ejecuta escaneos con perfil CIS
- **Política PCI**: Ejecuta escaneos con perfil PCI-DSS

**Configuración por defecto:**
- Namespace: `policies`
- Remediation Action: `enforce`
- Placement: Clusters con label `compliance: enabled` (definido inline en `policyDefaults`)

**Nota**: El Policy Generator crea automáticamente un PlacementRule basado en la configuración `placement` definida en `policyDefaults`. No se requiere un archivo de Placement separado.

### 2. Compliance Operator Installation

El archivo `compliance-operator.yaml` crea:

- **Namespace**: `openshift-compliance`
- **OperatorGroup**: Para el Compliance Operator
- **Subscription**: Suscripción al operador desde `redhat-operators`

**Canal**: `stable` (ajustable según tu entorno)

### 3. Scan Setting (`scan-setting.yaml`)

Configura escaneos automáticos con:

- **Schedule**: `0 1 * * *` (diario a la 1 AM)
- **Roles**: `master` y `worker`
- **Almacenamiento**:
  - Tamaño: `1Gi`
  - Rotación: Últimos 3 escaneos
  - Access Mode: `ReadWriteOnce`
- **Opciones**:
  - `maxRetryOnTimeout: 3`
  - `showNotApplicable: false`
  - `strictNodeScan: true`

### 4. Perfiles de Compliance

#### CIS Profile (`scansettingbinding-cis.yaml`)

Vincula los perfiles:
- `ocp4-cis` (control plane)
- `ocp4-cis-node` (nodos)

#### PCI-DSS Profile (`scansettingbinding-pci.yaml`)

Vincula los perfiles:
- `ocp4-pci-dss` (control plane)
- `ocp4-pci-dss-node` (nodos)

## ⚙️ Configuración

### Personalizar el Placement

El placement se define inline en `base/policy-generator-config.yaml` dentro de `policyDefaults`. Para cambiar qué clusters reciben las políticas, edita el selector:

```yaml
policyDefaults:
  placement:
    labelSelector:
      matchLabels:
        "compliance": "enabled"  # Cambia el label según tus necesidades
```

**Ejemplo**: Si quieres usar un label diferente:

```yaml
policyDefaults:
  placement:
    labelSelector:
      matchLabels:
        "environment": "production"
```

**Nota**: Si necesitas un Placement más complejo (con múltiples selectores, preferencias, etc.), puedes crear un archivo Placement separado y referenciarlo en las políticas individuales usando `placementRef`.

### Personalizar el Schedule

Edita `base/compliance-operator/scan-setting/scan-setting.yaml`:

```yaml
schedule: "0 1 * * *"  # Formato cron: minuto hora día mes día-semana
```

Ejemplos:
- `0 2 * * *` - Diario a las 2 AM
- `0 */6 * * *` - Cada 6 horas
- `0 0 * * 0` - Semanal los domingos a medianoche

### Habilitar/Deshabilitar Políticas

En `base/policy-generator-config.yaml`, cambia el campo `disabled`:

```yaml
policies:
  - name: run-cis-scan
    disabled: false  # true para deshabilitar
```

### Cambiar Remediation Action

Por defecto es `enforce`. Para cambiar a `inform` (solo reportar):

```yaml
policyDefaults:
  remediationAction: inform  # o 'enforce'
```

## 🚀 Instalación

### 1. Clonar el Repositorio

```bash
git clone <url-del-repositorio>
cd acm-policies/acm-policies
```

### 2. Etiquetar Clusters Objetivo

Etiqueta los clusters que deben recibir las políticas:

```bash
kubectl label managedcluster <nombre-cluster> compliance=enabled
```

**Nota**: Si personalizaste el selector en `policy-generator-config.yaml`, usa el label correspondiente.

### 3. Aplicar las Políticas

#### Opción A: Usando Kustomize directamente

```bash
kubectl apply -k base/
```

#### Opción B: Usando GitOps (ArgoCD/Flux)

1. Configura una aplicación en ArgoCD apuntando a este repositorio
2. Path: `base/`
3. ArgoCD aplicará automáticamente los cambios

### 4. Verificar la Instalación

```bash
# Verificar políticas creadas
kubectl get policies -n policies

# Verificar placement rules generados
kubectl get placementrule -n policies

# Verificar instalación del Compliance Operator
kubectl get subscription -n openshift-compliance

# Verificar escaneos configurados
kubectl get scansettingbinding -n openshift-compliance
```

## 📊 Uso

### Ver Estado de las Políticas

```bash
# Listar todas las políticas
kubectl get policies -n policies

# Ver detalles de una política
kubectl describe policy install-compliance-operator -n policies

# Ver compliance status
kubectl get policyreport -A
```

### Ver Resultados de Escaneos

```bash
# Ver escaneos ejecutados
kubectl get compliancescan -n openshift-compliance

# Ver resultados de escaneos
kubectl get compliancescanresult -n openshift-compliance

# Ver detalles de un escaneo específico
kubectl describe compliancescan <nombre-escaneo> -n openshift-compliance
```

### Verificar Compliance Status

```bash
# Ver el estado de compliance de los clusters
kubectl get compliancesuite -n openshift-compliance

# Ver remediaciones disponibles
kubectl get complianceremediation -n openshift-compliance
```

## 📋 Políticas Incluidas

| Política | Descripción | Remediation Action |
|----------|-------------|-------------------|
| `install-compliance-operator` | Instala el Compliance Operator | `enforce` |
| `scan-setting-periodic` | Configura escaneos periódicos | `enforce` |
| `run-cis-scan` | Ejecuta escaneos CIS | `enforce` |
| `run-pci-scan` | Ejecuta escaneos PCI-DSS | `enforce` |

## 🎯 Placement y Selectores

### Selector por Defecto

Las políticas usan el siguiente selector definido en `policyDefaults`:

```yaml
placement:
  labelSelector:
    matchLabels:
      compliance: enabled
```

El Policy Generator crea automáticamente un `PlacementRule` basado en esta configuración para cada política.

### Personalizar el Selector

Para cambiar qué clusters reciben las políticas:

1. **Opción 1 - Modificar el selector inline** (recomendado para casos simples):
   Edita `base/policy-generator-config.yaml` y cambia el label en `policyDefaults.placement`.

2. **Opción 2 - Placement separado** (para casos complejos):
   Si necesitas lógica más compleja (múltiples selectores, preferencias, etc.), puedes:
   - Crear un archivo Placement separado
   - Referenciarlo en políticas individuales usando `placementRef`

**Nota**: Asegúrate de que tus clusters tengan los labels apropiados antes de aplicar las políticas.

## 🔍 Troubleshooting

### Las políticas no se aplican a los clusters

1. Verifica que los clusters estén etiquetados:
   ```bash
   kubectl get managedcluster --show-labels
   ```

2. Verifica el placement rule generado:
   ```bash
   kubectl get placementrule -n policies
   kubectl describe placementrule <nombre-politica>-placement -n policies
   ```

3. Verifica el binding de políticas:
   ```bash
   kubectl get placementbinding -n policies
   ```

### El Compliance Operator no se instala

1. Verifica la suscripción:
   ```bash
   kubectl get subscription compliance-operator -n openshift-compliance
   ```

2. Revisa los eventos:
   ```bash
   kubectl get events -n openshift-compliance --sort-by='.lastTimestamp'
   ```

3. Verifica que el catálogo `redhat-operators` esté disponible:
   ```bash
   kubectl get catalogsource -n openshift-marketplace
   ```

### Los escaneos no se ejecutan

1. Verifica el ScanSetting:
   ```bash
   kubectl get scansetting periodic-daily -n openshift-compliance
   ```

2. Verifica el ScanSettingBinding:
   ```bash
   kubectl get scansettingbinding -n openshift-compliance
   ```

3. Revisa los logs del operador:
   ```bash
   kubectl logs -n openshift-compliance -l name=compliance-operator
   ```

### Problemas de almacenamiento

Si los escaneos fallan por falta de almacenamiento:

1. Verifica los PVCs:
   ```bash
   kubectl get pvc -n openshift-compliance
   ```

2. Ajusta el tamaño en `scan-setting.yaml`:
   ```yaml
   rawResultStorage:
     size: "2Gi"  # Aumenta según necesidad
   ```

## 📝 Notas Importantes

- **Namespace**: Todas las políticas se crean en el namespace `policies`
- **Compliance Operator**: Se instala en el namespace `openshift-compliance`
- **Rotación de resultados**: Solo se mantienen los últimos 3 escaneos para ahorrar espacio
- **Schedule**: Los escaneos se ejecutan diariamente a la 1 AM (configurable)
- **Remediation**: Las políticas están configuradas con `enforce`, lo que aplica automáticamente las remediaciones cuando es posible

## 🔗 Referencias

- [Red Hat Compliance Operator Documentation](https://docs.openshift.com/container-platform/latest/security/compliance_operator/compliance-operator-understanding.html)
- [ACM Policy Generator](https://github.com/stolostron/policy-generator)
- [Kustomize Documentation](https://kustomize.io/)
- [CIS Benchmark for OpenShift](https://www.cisecurity.org/benchmark/red_hat_openshift)
- [PCI-DSS Compliance](https://www.pcisecuritystandards.org/)

## 📄 Licencia

Este proyecto está configurado para uso interno. Ajusta según las políticas de tu organización.

---

**Mantenido por**: [Tu equipo/Organización]  
**Última actualización**: 2024
>>>>>>> 150bcca33d1bbabb2242a821aad1c7c4d43e8f50
