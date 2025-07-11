# Mejoras Para Implementar las POCs de Passkey / SPC con MasterCard y Visa

## Introducción

Secure Payment Confirmation (SPC) es una especificación W3C que extiende WebAuthn para casos de uso de pagos. SPC permite autenticación fuerte durante flujos de pago con funcionalidades adicionales específicas para el contexto de pagos.

**Documentación de referencia:**
- [W3C Secure Payment Confirmation Specification](https://www.w3.org/TR/secure-payment-confirmation/)
- [WebAuthn Level 3 Specification](https://www.w3.org/TR/webauthn-3/)

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

#### 4.2 Protección contra Ataques
- **Relay Attacks:** Validar contexto temporal del pago
- **Phishing:** Verificar información visual del instrumento de pago
- **Man-in-the-Middle:** Asegurar integridad de datos de pago

#### 4.3 Privacidad del Usuario
- Minimizar exposición de datos del instrumento de pago
- Implementar consentimiento explícito para uso cross-origin
- Auditoría de transacciones de pago

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




