# Mejoras Para Implementar las POCs de Passkey / SPC con MasterCard y Visa

## Introducción

Secure Payment Confirmation (SPC) es una especificación W3C que extiende WebAuthn para casos de uso de pagos. SPC permite autenticación fuerte durante flujos de pago con funcionalidades adicionales específicas para el contexto de pagos.

**Documentación de referencia:**
- [W3C Secure Payment Confirmation Specification](https://www.w3.org/TR/secure-payment-confirmation/)
- [WebAuthn Level 3 Specification](https://www.w3.org/TR/webauthn-3/)

## Fundamentos Cross-Origin en SPC

A continuación tienes una visión completa (y con ejemplos prácticos) de qué significa el campo crossOrigin en WebAuthn y en Secure Payment Confirmation (SPC), cómo lo calcula el navegador, por qué existe y qué debe hacer tu servidor cuando lo recibe.

**Resumen en una frase:**

crossOrigin es un indicador de riesgo: vale true cuando la llamada a navigator.credentials.create()/get() se hace desde un documento que NO es "same-origin" con la página superior; los navegadores lo añaden automáticamente al clientDataJSON, y los Relying Parties (RPs) lo usan para decidir si aceptar o rechazar la autenticación. En SPC, esta capacidad se amplía para que un tercero (por ejemplo, un comercio) pueda iniciar la autenticación de la passkey del banco, siempre que el RP verifique crossOrigin, topOrigin y otros campos de pago.

### 2.1 CrossOrigin en WebAuthn "puro"

#### 2.1.1 Dónde aparece

Cuando el navegador genera clientDataJSON, incluye cuatro campos fijos —type, challenge, origin y crossOrigin— y los escribe siempre en ese orden.

#### 2.1.2 Cómo se calcula

El valor se deriva del parámetro interno sameOriginWithAncestors:
- false → crossOrigin = false (llamada desde la propia página o un iframe del mismo origen).
- true → crossOrigin = true (llamada desde un iframe o ventana cuyo origen difiere del "top-level").

En la práctica, si tu widget de inicio de sesión está embebido en un iframe en el dominio del partner, el navegador marcará crossOrigin=true aunque el RP siga siendo tu dominio.

#### 2.1.3 Qué debe hacer el servidor
- Recupera y decodifica clientDataJSON.
- Comprueba que crossOrigin coincide con lo que esperabas para esa petición (por ejemplo, tu flujo permite iframes, o sólo aceptas same-origin). El algoritmo de verificación limitado del estándar incluye expresamente ese check.
- Si tu RP no admite autenticaciones en terceros dominios, rechaza el intento cuando crossOrigin=true.

Los desarrolladores lo ven clarísimo: en la estructura JS resultante, crossOrigin es un boolean que aparece junto al origin.
Por defecto es false; el propio GitHub del grupo explica que únicamente se añade (y vale true) en contextos realmente cross-origin.

### 2.2 Por qué existe (modelo de amenazas)

El campo protege contra ataques donde un sitio malicioso intenta reaprovechar la respuesta WebAuthn para suplantar al usuario en el dominio legítimo. Un RP puede:
1. Permitir sólo mismas-origin.
2. Permitir iframes de partners concretos, validando que origin y crossOrigin concuerdan con lo pactado.

Así corta intentos de "credential forwarding" si el atacante embebe tu login en un dominio diferente.

### 2.3 CrossOrigin dentro de Secure Payment Confirmation

#### 2.3.1 SPC habilita la autenticación por terceros

Una de las grandes diferencias de SPC es permitir que un comercio (o su PSP) invoque la passkey del banco sin redirigir al usuario. Esto se documenta como la "ceremonia de autenticación cross-origin".

#### 2.3.2 Campos extra que complementan crossOrigin

Para que el RP (el banco o la red de tarjetas) pueda verificar con seguridad:
- Se mantiene crossOrigin.
- Se añaden payment.topOrigin, payment.rpId, payment.payeeOrigin, etc.
- Además, el type cambia a "payment.get" —otra pista inequívoca de que la assertion viene de SPC y no de un login normal.

#### 2.3.3 Cómo se valida en SPC

El RP debe seguir los pasos de WebAuthn y además:
1. Confirmar type === "payment.get".
2. Asegurar que crossOrigin concuerda con el dominio que esperaba (por ejemplo, que la llamada vino realmente de merchant.com).
3. Verificar los campos de la extensión payment (importe, nombre del comercio, rpId, topOrigin…).

MDN resume el patrón: gracias a la permission policy "payment", un iframe cross-origin puede crear o usar la credencial; pero el RP debe validar todo lo anterior para evitar abusos.

### 2.4 Ejemplo de flujo cross-origin (SPC)

```
merchant.com (top) ─┐
                    │  ← iframe allow="payment"
   └─►   bank.com   ┘  (scripts del PSP)
```

1. bank.com pide PaymentRequest con method "secure-payment-confirmation"
2. Navegador muestra el diálogo SPC.
3. Usuario confirma con biometría → se genera `payment.get`, crossOrigin=true, topOrigin="https://merchant.com"
4. merchant.com recibe la assertion y la envía al banco / red
5. El RP valida firma, challenge, crossOrigin, topOrigin, etc. → SCA superada

Cada paso 3–5 se sustenta en los campos mencionados y en el chequeo de crossOrigin. Esta capacidad "invocar la passkey del banco sin salir del checkout" es lo que hace SPC atractivo para reducir fricción.

## Limitaciones Actuales de Ping 7.5.1

### 1. Falta de Soporte para el Campo "type" en ClientDataJSON

**Problema:** La implementación actual en `AuthenticationFlow.java` (línea 102) solo valida el tipo "webauthn.get":

```java
if (!("webauthn.get").equals(map.get("type"))) {
    logger.warn("client data type was incorrect, expecting webauth.get");
    return false;
}
```

**Requisito W3C:** Según la especificación SPC, el campo `type` en `CollectedClientData` debe ser:
- `"webauthn.get"` para autenticación WebAuthn tradicional
- `"payment.get"` para Secure Payment Confirmation

**Solución Propuesta:**
1. Modificar el método `AuthenticationFlow.accept()` para aceptar ambos tipos
2. Añadir un parámetro de configuración para especificar el modo SPC
3. Implementar validación condicional basada en el contexto de uso

### 2. Validaciones Adicionales Requeridas por SPC

**Según el estándar W3C, SPC requiere validaciones adicionales:**

#### 2.1 Validación de Información de Pago
- **Payment Instrument:** Validar datos del instrumento de pago (displayName, details, icon)
- **Payee Information:** Validar información del comerciante (payeeName, payeeOrigin)
- **Payment Amount:** Validar monto y moneda de la transacción

#### 2.2 Validación de Cross-Origin
- SPC permite autenticación cross-origin (el comerciante puede autenticar en nombre del banco)
- Requiere validación de `payeeOrigin` vs origen actual
- Implementar lista de orígenes permitidos para cada Relying Party

#### 2.3 Browser-Bound Keys
- **Propósito:** Proporcionar evidencia criptográfica de posesión del dispositivo
- **Requisito:** Claves auxiliares que residen únicamente en un dispositivo específico
- **Implementación:** Crear y gestionar pares de claves adicionales para evidencia de dispositivo

### 3. Modificaciones Requeridas en WebAuthnAuthenticationNode

#### 3.1 Soporte para Contexto SPC
**Cambios necesarios en `WebAuthnAuthenticationNode.java`:**

1. **Nueva Configuración:**
   - Añadir parámetro `enableSPC` en la configuración del nodo
   - Configurar `payeeOrigin` para validaciones cross-origin
   - Definir información del instrumento de pago

2. **Variables de Sesión:**
   - Almacenar información completa del flujo de autenticación
   - Guardar datos del instrumento de pago
   - Mantener contexto para página pivot

3. **Generación de Challenge:**
   - Incluir datos específicos de SPC en el challenge
   - Añadir información de pago al contexto del script

#### 3.2 Página Pivot para Comunicación Cross-Origin
**Requisitos:**
- Página intermedia para manejar respuesta del cliente
- Método JavaScript expuesto en el parent window
- Comunicación segura de datos de autenticación

**Implementación sugerida:**
```javascript
// En la página pivot
window.parent.postMessage({
    type: 'spc-response',
    credentialId: response.id,
    clientDataJSON: response.response.clientDataJSON,
    authenticatorData: response.response.authenticatorData,
    signature: response.response.signature
}, allowedOrigin);
```

### 4. Consideraciones de Seguridad

#### 4.1 Validaciones Cross-Origin
- Implementar lista blanca de orígenes permitidos
- Validar `payeeOrigin` contra orígenes registrados
- Verificar integridad de datos en comunicación cross-origin
- **No ignorar crossOrigin:** Es tan importante como challenge u origin
- **Registrar orígenes autorizados:** Validar que origin (WebAuthn) o topOrigin (SPC) coincide

#### 4.2 Protección contra Ataques
- **Relay Attacks:** Validar contexto temporal del pago
- **Phishing:** Verificar información visual del instrumento de pago
- **Man-in-the-Middle:** Asegurar integridad de datos de pago
- **Credential Forwarding:** Usar políticas de permisos adecuadas (publickey-credential-create, payment)

#### 4.3 Privacidad del Usuario
- Minimizar exposición de datos del instrumento de pago
- Implementar consentimiento explícito para uso cross-origin
- Auditoría de transacciones de pago

#### 4.4 Buenas Prácticas de Implementación
1. **Separar credenciales:** Distinguir entre credenciales de pago y login para evitar reutilización
2. **Validación completa:** Verificar todos los campos SPC (type, crossOrigin, topOrigin, payment fields)
3. **Políticas de seguridad:** Usar permission policies apropiadas para limitar acceso de iframes
4. **Monitoreo:** Vigilar cambios en navegadores (ej: Chrome 137 cambió excepciones de SecurityError a NotAllowedError)
5. **Pruebas cross-browser:** Validar compatibilidad en Chrome >= 95 y Edge

### 5. Integración con Payment Request API

SPC se integra con Payment Request API para proporcionar una experiencia de usuario completa:

```javascript
const request = new PaymentRequest([{
    supportedMethods: "secure-payment-confirmation",
    data: {
        credentialIds: [/* credential IDs */],
        challenge: new Uint8Array(/* challenge */),
        rpId: "bank.example",
        instrument: {
            displayName: "Visa ****1234",
            details: "****1234 | 01/29",
            icon: "https://bank.example/card-art.png"
        },
        payeeName: "Merchant Shop",
        payeeOrigin: "https://merchant.example"
    }
}], {
    total: {
        label: "Total",
        amount: { currency: "USD", value: "5.00" }
    }
});
```

## Plan de Implementación

### Fase 1: Soporte Básico SPC
1. Modificar `AuthenticationFlow.accept()` para soportar "payment.get"
2. Añadir configuración SPC en `WebAuthnAuthenticationNode`
3. Implementar validaciones básicas de información de pago

### Fase 2: Funcionalidades Avanzadas
1. Implementar soporte para browser-bound keys
2. Desarrollar página pivot para comunicación cross-origin
3. Añadir validaciones de seguridad específicas de SPC

### Fase 3: Integración Completa
1. Integrar con Payment Request API
2. Implementar auditoría y logging específico de SPC
3. Pruebas de integración con MasterCard y Visa

## Archivos a Modificar

1. **`AuthenticationFlow.java`:** Validación de tipo de cliente
2. **`WebAuthnAuthenticationNode.java`:** Configuración y contexto SPC
3. **Scripts de cliente:** Soporte para Payment Request API
4. **Recursos web:** Página pivot para comunicación cross-origin
5. **Configuración:** Nuevos parámetros de configuración SPC