# ACM Policies - Compliance Operator para OpenShift

Este repositorio contiene políticas de Advanced Cluster Management (ACM) para desplegar y configurar el Compliance Operator de Red Hat OpenShift, permitiendo realizar escaneos de cumplimiento (compliance) automatizados en clusters OpenShift.

## Descripción General

El proyecto utiliza **Kustomize** y el **Policy Generator** de ACM para gestionar políticas que:

1. **Instalan el Compliance Operator** en clusters OpenShift
2. **Configuran escaneos periódicos** de compliance
3. **Ejecutan perfiles de compliance** (CIS y PCI-DSS) de forma automatizada

## Estructura del Proyecto

```
acm-policies/
├── base/
│   ├── kustomization.yaml              # Configuración principal de Kustomize
│   ├── policy-generator-config.yaml    # Configuración del generador de políticas
│   └── compliance-operator/
│       ├── install/
│       │   └── compliance-operator.yaml # Instalación del operador
│       ├── scan-setting/
│       │   └── scan-setting.yaml       # Configuración de escaneos periódicos
│       └── profiles/
│           ├── cis/
│           │   └── scansettingbinding-cis.yaml    # Binding para perfil CIS
│           └── pci/
│               └── scansettingbinding-pci.yaml   # Binding para perfil PCI-DSS
└── README.md
```

## Componentes Principales

### 1. Kustomization (`base/kustomization.yaml`)

Archivo raíz que configura Kustomize para generar las políticas ACM usando el Policy Generator.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
  - policy-generator-config.yaml
```

### 2. Policy Generator Config (`base/policy-generator-config.yaml`)

Define las políticas que se generarán y aplicarán a los clusters. Contiene:

- **Policy Defaults**: Configuración por defecto para todas las políticas
  - Namespace: `policies`
  - Remediation Action: `enforce` (aplica cambios automáticamente)
  - Placement: Selecciona clusters con label `compliance: enabled`

- **Políticas Generadas**:
  1. **install-compliance-operator**: Instala el Compliance Operator
  2. **scan-setting-periodic**: Configura el schedule de escaneos
  3. **run-cis-scan**: Ejecuta escaneos CIS (Center for Internet Security)
  4. **run-pci-scan**: Ejecuta escaneos PCI-DSS (Payment Card Industry)

### 3. Compliance Operator Installation (`base/compliance-operator/install/compliance-operator.yaml`)

Crea los recursos necesarios para instalar el Compliance Operator:

- **Namespace**: `openshift-compliance`
- **OperatorGroup**: Define el alcance del operador
- **Subscription**: Suscripción al operador desde `redhat-operators` (canal `stable`)

### 4. Scan Setting (`base/compliance-operator/scan-setting/scan-setting.yaml`)

Configura los escaneos periódicos:

- **Schedule**: `0 1 * * *` (diario a la 1:00 AM)
- **Roles**: Escanea nodos `master` y `worker`
- **Almacenamiento**:
  - Tamaño: 1Gi
  - Rotación: Mantiene los últimos 3 escaneos
- **Configuración adicional**: Reintentos, modo estricto, etc.

### 5. Perfiles de Compliance

#### CIS Profile (`base/compliance-operator/profiles/cis/scansettingbinding-cis.yaml`)

Vincula los perfiles CIS al ScanSetting:
- `ocp4-cis-1-7`: Perfil CIS versión 1.7 para el cluster
- `ocp4-cis-node-1-7`: Perfil CIS versión 1.7 para los nodos

#### PCI-DSS Profile (`base/compliance-operator/profiles/pci/scansettingbinding-pci.yaml`)

Vincula los perfiles PCI-DSS al ScanSetting:
- `ocp4-pci-dss-4-0`: Perfil PCI-DSS versión 4.0 para el cluster
- `ocp4-pci-dss-node-4-0`: Perfil PCI-DSS versión 4.0 para los nodos

## Uso

### Prerrequisitos

1. Cluster OpenShift con ACM (Advanced Cluster Management) instalado
2. Acceso al catálogo de operadores de Red Hat (`redhat-operators`)
3. Clusters target etiquetados con `compliance: enabled`

### Aplicar las Políticas

1. Etiquetar los clusters donde se aplicarán las políticas:
```bash
kubectl label managedcluster <nombre-cluster> compliance=enabled
```

2. Generar y aplicar las políticas usando Kustomize:
```bash
kubectl apply -k base/
```

O si usas GitOps con ArgoCD/Flux, este repositorio puede ser referenciado directamente.

## Configuración

### Modificar el Schedule de Escaneos

Edita `base/compliance-operator/scan-setting/scan-setting.yaml`:

```yaml
schedule: "0 1 * * *"  # Formato cron: minuto hora día mes día-semana
```

### Habilitar/Deshabilitar Políticas

En `base/policy-generator-config.yaml`, cambia el campo `disabled`:

```yaml
- name: run-cis-scan
  disabled: false  # true para deshabilitar
```

### Cambiar el Placement

Modifica el `labelSelector` en `policyDefaults.placement`:

```yaml
placement:
  labelSelector:
    matchLabels:
      "compliance": "enabled"  # Cambia según tus necesidades
```

## Notas Importantes

1. **Remediation Action**: Está configurado como `enforce`, lo que significa que las políticas aplicarán cambios automáticamente. Si prefieres solo monitorear, cambia a `inform`.

2. **Perfiles de Compliance**: Verifica que los perfiles específicos estén disponibles en tu cluster:
   - `ocp4-cis-1-7` y `ocp4-cis-node-1-7` (CIS)
   - `ocp4-pci-dss-4-0` y `ocp4-pci-dss-node-4-0` (PCI-DSS)

   Los perfiles PCI-DSS pueden requerir suscripciones adicionales. Puedes verificar los perfiles disponibles con:
   ```bash
   kubectl get profiles -n openshift-compliance
   ```

3. **Almacenamiento**: Los escaneos generan resultados que se almacenan en PVs. Asegúrate de tener suficiente espacio de almacenamiento.

4. **Canal del Operador**: El canal `stable` puede variar según la versión de OpenShift. Verifica el canal correcto en tu entorno.

## Verificación

Para verificar que las políticas se aplicaron correctamente:

```bash
# Ver políticas generadas
kubectl get policies -n policies

# Ver estado del Compliance Operator
kubectl get pods -n openshift-compliance

# Ver escaneos programados
kubectl get scansettingbindings -n openshift-compliance

# Ver perfiles de compliance disponibles
kubectl get profiles -n openshift-compliance

# Ver resultados de escaneos
kubectl get compliancescans -n openshift-compliance
```

