// ==============================================================================
//  PROYECTO: EVENTO LUMINATE / EXTIÉNDETE — BACKEND OFICIAL FACTURACIÓN
//  Archivo: Codigo_gs_Facturacion.gs
//  Arquitectura y Ciberseguridad: Code Ahumada
//
//  CARACTERÍSTICAS Y BLINDAJE:
//  - COMPROBANTE PDF 100% UNIFICADO (Idéntico al diseño oficial de la web con QR)
//  - Checkout Atómico (1 sola llamada Web -> Google Sheets + Mercado Pago)
//  - Webhook IPN de Mercado Pago con auto-confirmación
//  - Emisión de Comprobante HTML automático al comprador
//  - Adjunto automático de Comprobante Oficial en formato PDF
//  - Helper estricto yaSeEnvioEmail (resuelve bug de subcadena NO ENVIADO)
//  - Blindaje Anti-Bucle: Columna N (Control de Email) + CacheService (6 horas)
//  - Auto-rescate de pagos aprobados sin email enviado
//  - Función REENVIAR_COMPROBANTES_PENDIENTES para rescate manual en 1 clic
//  - Zero-Client Secrets (Token de MP y Resend en ScriptProperties)
//  - Blindaje contra Formula Injection (sanitizarFormula en todas las entradas)
//  - Concurrencia segura con LockService (evita registros duplicados)
//  - 100% LIBRE DE TELEGRAM (Acreditación delegada al panel web oficial)
// ==============================================================================

// --- CONFIGURACIÓN AISLADA EN EL SERVIDOR (ZERO-CLIENT SECRETS) ---
function getMercadoPagoToken() {
  const prop = PropertiesService.getScriptProperties().getProperty("MP_ACCESS_TOKEN");
  if (prop && prop.trim() !== "") return prop.trim();
  const fallback = PropertiesService.getScriptProperties().getProperty("MERCADOPAGO_TOKEN") || PropertiesService.getScriptProperties().getProperty("MP_TOKEN");
  if (fallback && fallback.trim() !== "") return fallback.trim();
  return "";
}

function getResendApiKey() {
  return PropertiesService.getScriptProperties().getProperty("RESEND_API_KEY") || "";
}

const SPREADSHEET_ID      = "1dCTBmZTsOkgf1evvLCVkvCSWOr_-gYs6yXln1kxTYAM";
const HOJA_PAGO_ONLINE    = "Pago-Online";
const SECURITY_TOKEN      = "GRAN_REY_SECURE_2026";

// ⚠️ PRECIO OFICIAL DE LA ENTRADA (Inmutable en el servidor: $29.990 ARS)
const PRECIO_ENTRADA      = 29990; 

// 📧 CONFIGURACIÓN DE CORREO OFICIAL (RESEND)
const EMAIL_FACTURACION   = "facturacion@entradas.iglesiagranrey.com";
const NOMBRE_REMITENTE    = "Iglesia Gran Rey · Facturación";
const EMAIL_RESPUESTAS    = "cris.ahu777@gmail.com"; // Casilla para recibir dudas o respuestas de compradores

// --- HELPERS GLOBALES ---
/**
 * Helper de respuesta JSON estricta (ContentService con MIME JSON)
 */
function jsonOutput(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

/**
 * Sanitizar entradas para prevenir inyección de fórmulas en Google Sheets (Formula Injection)
 */
function sanitizarFormula(val) {
  if (val === null || val === undefined) return "";
  let str = String(val).trim();
  if (/^[=+\-@\t\r]/.test(str)) {
    return "'" + str;
  }
  return str;
}

/**
 * Validación estricta para saber si el email ya fue efectivamente despachado.
 */
function yaSeEnvioEmail(val) {
  const s = String(val || "").trim().toUpperCase();
  return s.startsWith("ENVIADO ✅") || s === "ENVIADO" || s === "SI";
}

// --- ENRUTAMIENTO GET ---
function doGet(e) {
  try {
    if (!e || !e.parameter) {
      return jsonOutput({ ok: false, error: "Acceso denegado: parámetros no provistos" });
    }

    // 1. Soporte para Webhook IPN de Mercado Pago por GET
    if (e.parameter.topic === "payment" || e.parameter.type === "payment" || e.parameter["data.id"] || e.parameter.id) {
      const paymentId = e.parameter["data.id"] || e.parameter.id;
      if (paymentId) procesarPagoMercadoPago(paymentId);
      return ContentService.createTextOutput("OK");
    }

    // 2. Consulta pública y segura de comprobante oficial por referencia única
    if (e.parameter.action === "consultar_comprobante" && e.parameter.ref) {
      return jsonOutput(consultarDatosComprobante(e.parameter.ref));
    }

    // 3. Health check con token de seguridad
    if (e.parameter.token !== SECURITY_TOKEN) {
      return jsonOutput({ ok: false, error: "Acceso denegado: token inválido" });
    }

    const mpTokenConfigured = !!getMercadoPagoToken();
    const resendConfigured = !!getResendApiKey();

    return jsonOutput({
      ok: true,
      status: "ONLINE",
      message: "✅ SCRIPT FACTURACIÓN ONLINE Y OPERATIVO (SIN TELEGRAM)",
      precioOficial: PRECIO_ENTRADA,
      emailAuditoria: EMAIL_FACTURACION,
      mpTokenConfigurado: mpTokenConfigured,
      resendConfigurado: resendConfigured,
      timestamp: new Date().toISOString()
    });
  } catch (err) {
    console.error("Error en doGet:", err);
    return jsonOutput({ ok: false, error: "ERROR: " + err.toString() });
  }
}

/**
 * Consulta los datos oficiales de una compra en la hoja Pago-Online mediante su referencia única.
 * Permite a la página web (registro-exitoso.html) mostrar los datos fidedignos del comprador y sus asistentes.
 */
function consultarDatosComprobante(ref) {
  const cleanRef = String(ref || "").trim();
  if (!cleanRef) {
    return { ok: false, error: "Referencia no válida o vacía." };
  }

  try {
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const sheet = ss.getSheetByName(HOJA_PAGO_ONLINE);
    if (!sheet) {
      return { ok: false, error: "Hoja de registros no encontrada." };
    }

    const data = sheet.getDataRange().getValues();
    const asistentes = [];
    let buyerEmail = "";
    let buyerPhone = "";
    let city = "";
    let pastor = "";
    let paymentId = "";
    let estadoPago = "PENDIENTE";
    let fecha = "";
    let totalCalculado = 0;

    for (let i = 1; i < data.length; i++) {
      const rowRef = String(data[i][11] || "").trim(); // Columna L: External Reference
      if (rowRef === cleanRef) {
        const nombreAsistente = String(data[i][2] || "").trim(); // Columna C: Nombre
        if (nombreAsistente) asistentes.push(nombreAsistente);

        if (!fecha) fecha = String(data[i][1] || "").trim(); // Columna B: Fecha
        if (!city) city = String(data[i][3] || "").trim(); // Columna D: Ciudad/Iglesia
        if (!paymentId) paymentId = String(data[i][5] || "").trim(); // Columna F: ID Mercado Pago
        if (!pastor) pastor = String(data[i][9] || "").trim(); // Columna J: Pastor

        const contacto = String(data[i][10] || "").trim(); // Columna K: Contacto
        if (contacto.includes("|")) {
          const parts = contacto.split("|");
          if (!buyerEmail) buyerEmail = parts[0].trim();
          if (!buyerPhone) buyerPhone = parts[1].trim();
        } else if (contacto.includes("@") && !buyerEmail) {
          buyerEmail = contacto;
        }

        const est = String(data[i][12] || "").trim(); // Columna M: Estado Pago
        if (est.toUpperCase().includes("APROBADO")) {
          estadoPago = "PAGO APROBADO ✅";
        }

        const montoRow = Number(data[i][4]); // Columna E: Monto
        totalCalculado += (!isNaN(montoRow) && montoRow > 0) ? montoRow : PRECIO_ENTRADA;
      }
    }

    if (asistentes.length === 0) {
      return {
        ok: false,
        found: false,
        error: "No se encontró ninguna compra asociada al código de referencia: " + cleanRef
      };
    }

    return {
      ok: true,
      found: true,
      compra: {
        external_reference: cleanRef,
        paymentId: paymentId,
        buyerEmail: buyerEmail,
        buyerPhone: buyerPhone,
        city: city,
        pastor: pastor,
        totalPrice: totalCalculado,
        quantity: asistentes.length,
        names: asistentes,
        estadoPago: estadoPago,
        fecha: fecha
      }
    };
  } catch (err) {
    console.error("Error al consultar comprobante:", err);
    return { ok: false, error: "Error al recuperar datos: " + err.toString() };
  }
}

// --- ENRUTAMIENTO POST ---
function doPost(e) {
  try {
    if (!e || !e.postData || !e.postData.contents) {
      return jsonOutput({ ok: false, error: "Sin datos en el cuerpo de la petición" });
    }

    let contents;
    try {
      contents = JSON.parse(e.postData.contents);
    } catch (parseErr) {
      return jsonOutput({ ok: false, error: "JSON_INVALID: Formato JSON no válido" });
    }

    // 1. Webhook de Mercado Pago (IPN / Webhook POST)
    if (contents.type === "payment" || (contents.action && contents.action.includes("payment"))) {
      return handleMercadoPagoWebhook(contents);
    }

    // 2. Checkout Atómico (Web oficial: Guarda en Sheets + Crea preferencia MP en 1 sola llamada)
    if (contents.action === "checkout_atomico") {
      if (contents.token !== SECURITY_TOKEN) {
        return jsonOutput({ ok: false, error: "No autorizado: token inválido" });
      }
      return ejecutarCheckoutAtomico(contents.data);
    }

    // 3. Creación de preferencia (Compatibilidad anterior, blindada con precio en servidor)
    if (contents.action === "create_preference") {
      if (contents.token !== SECURITY_TOKEN) {
        return jsonOutput({ ok: false, error: "No autorizado: token inválido" });
      }
      return createPreference(contents.data);
    }

    // 4. Registro previo en Sheets (Compatibilidad anterior)
    if (contents.rows) {
      if (contents.token !== SECURITY_TOKEN) {
        return jsonOutput({ ok: false, error: "No autorizado: token inválido" });
      }
      return handleWebRegistration(contents.rows, contents.external_reference);
    }

    return jsonOutput({ ok: false, error: "ACCION_DESCONOCIDA" });
  } catch (error) {
    console.error("Error en doPost:", error.toString());
    return jsonOutput({ ok: false, error: "ERROR: " + error.toString() });
  }
}

// --- CHECKOUT ATÓMICO (GUARDADO EN LOTE + PREFERENCIA MP) ---
function ejecutarCheckoutAtomico(data) {
  if (!data) {
    return jsonOutput({ ok: false, error: "Datos de checkout no provistos." });
  }

  const lock = LockService.getScriptLock();
  try {
    lock.waitLock(15000);

    const names = Array.isArray(data.names) ? data.names : [];
    const qIndiv = Number(data.quantities?.individual || data.quantity || (names.length > 0 ? names.length : 1));
    if (isNaN(qIndiv) || qIndiv <= 0) {
      return jsonOutput({ ok: false, error: "Cantidad inválida de entradas." });
    }

    const totalCalculado = qIndiv * PRECIO_ENTRADA;
    const externalRef = sanitizarFormula(data.external_reference || ("LUM-" + Date.now() + "-" + Math.floor(1000 + Math.random() * 9000)));
    const nowStr = Utilities.formatDate(new Date(), "GMT-3", "dd/MM/yyyy HH:mm");

    const buyerEmail = sanitizarFormula(data.buyerEmail || "");
    const buyerPhone = sanitizarFormula(data.buyerPhone || "");
    const contactoStr = buyerEmail + (buyerPhone ? " | " + buyerPhone : "");
    const city = sanitizarFormula(data.city || "");
    const pastor = sanitizarFormula(data.pastor || "");
    const tipoLabel = sanitizarFormula(data.typeLabel || "Individual / General");

    const batchRows = [];
    for (let i = 0; i < qIndiv; i++) {
      const nombreAsistente = sanitizarFormula(names[i] || ("Asistente #" + (i + 1)));
      batchRows.push([
        tipoLabel,       // Col 1 (A): Entrada
        nowStr,          // Col 2 (B): Fecha de pago
        nombreAsistente, // Col 3 (C): Nombre de la persona
        city,            // Col 4 (D): Ciudad / Iglesia
        PRECIO_ENTRADA,  // Col 5 (E): Monto que pagó
        "",              // Col 6 (F): ID Mercado Pago
        "NO",            // Col 7 (G): Pulsera entregada
        "",              // Col 8 (H): Responsable entrega
        "",              // Col 9 (I): Fecha entrega
        pastor,          // Col 10 (J): Pastor
        contactoStr,     // Col 11 (K): Contacto Comprador (Email | Teléfono)
        externalRef,     // Col 12 (L): External Reference
        "PENDIENTE",     // Col 13 (M): Estado Pago
        "PENDIENTE"      // Col 14 (N): Comprobante Enviado
      ]);
    }

    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const sheet = ss.getSheetByName(HOJA_PAGO_ONLINE);
    if (!sheet) {
      return jsonOutput({ ok: false, error: "Hoja '" + HOJA_PAGO_ONLINE + "' no encontrada en la planilla." });
    }

    const headerCol14 = sheet.getRange(1, 14).getValue();
    if (!headerCol14 || String(headerCol14).trim() === "") {
      sheet.getRange(1, 14).setValue("Email Comprobante");
    }

    const lastRow = sheet.getLastRow();
    sheet.getRange(lastRow + 1, 1, batchRows.length, batchRows[0].length).setValues(batchRows);
    SpreadsheetApp.flush();

    const prefResult = invocarMercadoPago(externalRef, tipoLabel, totalCalculado, qIndiv, buyerEmail, buyerPhone);
    return jsonOutput(prefResult);

  } catch (err) {
    console.error("Error en checkoutAtomico:", err);
    return jsonOutput({ ok: false, error: "Error al procesar pedido: " + err.toString() });
  } finally {
    lock.releaseLock();
  }
}

// --- CREAR PREFERENCIA MERCADO PAGO ---
function invocarMercadoPago(externalRef, typeLabel, totalAmount, quantity, buyerEmail, buyerPhone) {
  const mpToken = getMercadoPagoToken();
  if (!mpToken) {
    console.error("Error: MP_ACCESS_TOKEN no está configurado en ScriptProperties.");
    return {
      ok: false,
      error: "El token de Mercado Pago (MP_ACCESS_TOKEN) no está configurado en las ScriptProperties del servidor.",
      details: "Por favor agregue la propiedad 'MP_ACCESS_TOKEN' en Configuración del Proyecto > Propiedades de la secuencia de comandos en Google Apps Script."
    };
  }

  const url = "https://api.mercadopago.com/checkout/preferences";
  const q = quantity || 1;
  const itemTitle = q > 1 ? ("Entradas EVENTO LUMINATE x " + q) : "Entrada EVENTO LUMINATE";

  const payload = {
    items: [{
      title: itemTitle,
      quantity: 1,
      unit_price: Number(totalAmount),
      currency_id: "ARS"
    }],
    external_reference: String(externalRef),
    back_urls: {
      success: "https://luminate.iglesiagranrey.com/registro-exitoso.html",
      failure: "https://luminate.iglesiagranrey.com/"
    },
    auto_return: "approved"
  };

  try {
    const serviceUrl = ScriptApp.getService().getUrl();
    if (serviceUrl && serviceUrl.startsWith("https://")) {
      payload.notification_url = serviceUrl;
    }
  } catch (svcErr) {
    console.warn("No se pudo obtener la URL de servicio automáticamente:", svcErr);
  }

  if (buyerEmail && String(buyerEmail).includes("@")) {
    payload.payer = {
      email: String(buyerEmail).trim()
    };
  }

  const options = {
    method: "post",
    contentType: "application/json",
    headers: { Authorization: "Bearer " + mpToken },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch(url, options);
    const responseCode = response.getResponseCode();
    const responseText = response.getContentText();

    let prefData;
    try {
      prefData = JSON.parse(responseText);
    } catch (parseErr) {
      console.error("Respuesta no parseable de MP (" + responseCode + "): " + responseText);
      return {
        ok: false,
        error: "Mercado Pago devolvió una respuesta no válida (HTTP " + responseCode + ")",
        raw: responseText
      };
    }

    if (responseCode === 200 || responseCode === 201) {
      if (prefData && prefData.init_point) {
        return {
          ok: true,
          success: true,
          init_point: prefData.init_point,
          id: prefData.id,
          external_reference: externalRef
        };
      }
    }

    console.error("Error MP HTTP " + responseCode + ":", responseText);
    let detalleError = "No se pudo generar el link de pago en Mercado Pago (HTTP " + responseCode + ").";
    if (prefData && prefData.message) detalleError += " " + prefData.message;
    else if (prefData && prefData.error) detalleError += " " + prefData.error;

    return {
      ok: false,
      error: detalleError,
      statusCode: responseCode,
      details: prefData
    };

  } catch (fetchErr) {
    console.error("Excepción de red al contactar Mercado Pago:", fetchErr);
    return {
      ok: false,
      error: "Error de red al conectar con Mercado Pago: " + fetchErr.toString()
    };
  }
}

// --- FUNCIONES DE COMPATIBILIDAD ANTERIOR ---
function createPreference(data) {
  if (!data) return jsonOutput({ ok: false, error: "Sin datos" });
  const q = Number(data.quantity || 1);
  const total = (q > 0 ? q : 1) * PRECIO_ENTRADA;
  const res = invocarMercadoPago(data.external_reference, data.typeLabel, total, q, data.buyerEmail, data.buyerPhone);
  return jsonOutput(res);
}

function handleWebRegistration(rows, externalRef) {
  if (!Array.isArray(rows) || rows.length === 0) {
    return jsonOutput({ ok: false, error: "Filas vacías o inválidas" });
  }

  const lock = LockService.getScriptLock();
  try {
    lock.waitLock(15000);
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const sheet = ss.getSheetByName(HOJA_PAGO_ONLINE);
    if (!sheet) return jsonOutput({ ok: false, error: "Hoja Pago-Online no encontrada" });

    const headerCol14 = sheet.getRange(1, 14).getValue();
    if (!headerCol14 || String(headerCol14).trim() === "") {
      sheet.getRange(1, 14).setValue("Email Comprobante");
    }

    const batch = rows.map(function(row) {
      return [
        sanitizarFormula(row.tipo || "Individual"),
        sanitizarFormula(row.fecha || Utilities.formatDate(new Date(), "GMT-3", "dd/MM/yyyy HH:mm")),
        sanitizarFormula(row.nombre),
        sanitizarFormula(row.ciudad),
        PRECIO_ENTRADA,
        "",
        "NO",
        "",
        "",
        sanitizarFormula(row.pastor || ""),
        sanitizarFormula(row.contacto || ""),
        sanitizarFormula(externalRef),
        "PENDIENTE",
        "PENDIENTE"
      ];
    });

    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, batch.length, batch[0].length).setValues(batch);
    SpreadsheetApp.flush();

    return jsonOutput({ ok: true, message: "OK" });
  } catch (err) {
    return jsonOutput({ ok: false, error: err.toString() });
  } finally {
    lock.releaseLock();
  }
}

// --- WEBHOOK Y CONFIRMACIÓN DE PAGO MERCADO PAGO ---
function handleMercadoPagoWebhook(contents) {
  const paymentId = contents.data ? contents.data.id : contents.id;
  if (!paymentId) return ContentService.createTextOutput("SIN_ID");
  procesarPagoMercadoPago(paymentId);
  return ContentService.createTextOutput("OK");
}

function procesarPagoMercadoPago(paymentId) {
  try {
    const mpToken = getMercadoPagoToken();
    if (!mpToken) {
      console.error("No se puede procesar pago " + paymentId + ": MP_ACCESS_TOKEN no configurado.");
      return;
    }

    const url = "https://api.mercadopago.com/v1/payments/" + paymentId;
    const options = {
      headers: { Authorization: "Bearer " + mpToken },
      muteHttpExceptions: true
    };
    const response = UrlFetchApp.fetch(url, options);
    if (response.getResponseCode() !== 200) {
      console.warn("Mercado Pago devolvió HTTP " + response.getResponseCode() + " para el pago " + paymentId);
      return;
    }

    const payment = JSON.parse(response.getContentText());

    // 🛑 REGLA INVIOLABLE DE CRISTIAN:
    // Solo si el pago está ESTRICTAMENTE aprobado en Mercado Pago se procede.
    if (payment.status === "approved" && payment.external_reference) {
      confirmPaymentInSheet(payment.external_reference, String(paymentId), payment);
    } else {
      console.log("Pago " + paymentId + " no aprobado (estado: " + payment.status + "). No se envía comprobante ni se altera la planilla.");
    }
  } catch (err) {
    console.error("Error al procesar pago " + paymentId + ": ", err);
  }
}

// --- CONFIRMACIÓN EN PLANILLA Y DISPARO CONTROLADO DE CORREO ---
function confirmPaymentInSheet(ref, paymentId, paymentObj) {
  const cleanRef = String(ref || "").trim();
  if (!cleanRef) return;

  const cache = CacheService.getScriptCache();
  const cacheKey = "mail_sent_" + cleanRef;

  // Solo bloquear si la caché dice que el mail YA fue enviado previamente
  if (cache.get(cacheKey) === "true") {
    console.log("Comprobante ya enviado (detectado por CacheService) para ref: " + cleanRef + ". Se omite procesamiento.");
    return;
  }

  const lock = LockService.getScriptLock();
  try {
    lock.waitLock(15000);
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const sheet = ss.getSheetByName(HOJA_PAGO_ONLINE);
    if (!sheet) {
      console.error("Hoja Pago-Online no encontrada al confirmar pago.");
      return;
    }

    const headerCol14 = sheet.getRange(1, 14).getValue();
    if (!headerCol14 || String(headerCol14).trim() === "") {
      sheet.getRange(1, 14).setValue("Email Comprobante");
    }

    const data = sheet.getDataRange().getValues();
    let modificado = false;
    let debeEnviarEmail = false;
    let compradorEmail = "";
    let compradorPhone = "";
    let ciudadGrupo = "";
    let pastorGrupo = "";
    const asistentes = [];
    let totalMonto = 0;
    const nowStr = Utilities.formatDate(new Date(), "GMT-3", "dd/MM/yyyy HH:mm");

    for (let i = 1; i < data.length; i++) {
      if (String(data[i][11]).trim() === cleanRef) {
        const estadoPagoActual = String(data[i][12] || "").trim();
        const emailYaEnviado = yaSeEnvioEmail(data[i][13]);

        // 1. Si no estaba aprobado en la planilla, lo aprobamos
        if (estadoPagoActual !== "PAGO APROBADO ✅") {
          data[i][5] = String(paymentId);      // Columna F: ID de Pago Mercado Pago
          data[i][12] = "PAGO APROBADO ✅";    // Columna M: Estado Pago Exitoso
          modificado = true;

          if (!emailYaEnviado) {
            debeEnviarEmail = true;
            data[i][13] = "ENVIADO ✅ " + nowStr;
          }
        } else {
          // 2. Si ya estaba aprobado pero el email NUNCA se había enviado:
          if (!emailYaEnviado) {
            debeEnviarEmail = true;
            data[i][13] = "ENVIADO ✅ " + nowStr;
            modificado = true;
          } else {
            debeEnviarEmail = false;
          }
        }

        asistentes.push(String(data[i][2]));
        ciudadGrupo = String(data[i][3]);
        totalMonto += Number(data[i][4]) || PRECIO_ENTRADA;
        pastorGrupo = String(data[i][9]);

        // Extraer email y teléfono de la columna 11 (K)
        const contacto = String(data[i][10] || "");
        if (contacto.includes("|")) {
          const parts = contacto.split("|");
          if (!compradorEmail) compradorEmail = parts[0].trim();
          if (!compradorPhone) compradorPhone = parts[1].trim();
        } else if (contacto.includes("@") && !compradorEmail) {
          compradorEmail = contacto.trim();
        }
      }
    }

    if (modificado) {
      sheet.getDataRange().setValues(data);
      SpreadsheetApp.flush();
    }

    // 🛑 CANDADO 2: Solo despachar si debeEnviarEmail es TRUE
    if (debeEnviarEmail) {
      cache.put(cacheKey, "true", 21600); // 6 horas

      if (!compradorEmail && paymentObj && paymentObj.payer && paymentObj.payer.email) {
        compradorEmail = paymentObj.payer.email;
      }

      if (compradorEmail && String(compradorEmail).includes("@")) {
        enviarComprobantePorEmail(compradorEmail, compradorPhone, cleanRef, paymentId, asistentes, totalMonto, ciudadGrupo, pastorGrupo);
      } else {
        console.warn("No se pudo obtener el email del comprador para ref: " + cleanRef);
      }
    } else {
      console.log("Comprobante ya enviado previamente para ref: " + cleanRef + ". No se reenvía ningún correo.");
    }

  } catch (err) {
    console.error("Error confirmPaymentInSheet:", err);
  } finally {
    lock.releaseLock();
  }
}

// --- FUNCIÓN MANUAL PARA RESCATAR Y ENVIAR COMPROBANTES PENDIENTES ---
function REENVIAR_COMPROBANTES_PENDIENTES() {
  const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
  const sheet = ss.getSheetByName(HOJA_PAGO_ONLINE);
  if (!sheet) {
    Logger.log("Error: Hoja Pago-Online no encontrada.");
    return;
  }

  const data = sheet.getDataRange().getValues();
  const cache = CacheService.getScriptCache();
  let enviados = 0;

  for (let i = 1; i < data.length; i++) {
    const estadoPago = String(data[i][12] || "").trim();
    const ref = String(data[i][11] || "").trim();
    const paymentId = String(data[i][5] || "").trim();
    const emailEstado = data[i][13];

    if (estadoPago === "PAGO APROBADO ✅" && !yaSeEnvioEmail(emailEstado) && ref) {
      Logger.log("Enviando comprobante oficial para Fila " + (i + 1) + " (Ref: " + ref + ")...");
      cache.remove("mail_sent_" + ref); // Limpiar candado previo
      confirmPaymentInSheet(ref, paymentId, null);
      enviados++;
    }
  }

  Logger.log("Proceso terminado. Comprobantes enviados: " + enviados);
}

// ==============================================================================
//  PLANTILLA HTML PARA GENERACIÓN DE PDF OFICIAL UNIFICADO
//  100% IDÉNTICO AL DISEÑO DE LA WEB CON QR CODE INCORPORADO
// ==============================================================================
function generarHtmlComprobantePdf(ref, paymentId, asistentes, totalMonto, ciudad, pastor, buyerEmail, buyerPhone) {
  const fechaObj = new Date();
  const fechaStr = Utilities.formatDate(fechaObj, "GMT-3", "dd/MM/yyyy");
  const horaStr = Utilities.formatDate(fechaObj, "GMT-3", "hh:mm a").toLowerCase();
  const totalFormatted = Number(totalMonto).toLocaleString('es-AR');
  const cant = asistentes.length || 1;
  const precioUnit = Math.round(totalMonto / cant);
  const precioUnitFormatted = Number(precioUnit).toLocaleString('es-AR');

  // 1. Obtener imagen de Código QR oficial para incrustar en el PDF
  let qrImgTag = "";
  try {
    const qrPayload = encodeURIComponent("EVENTO LUMINATE 2026 | REF: " + ref + " | COMPRADOR: " + (buyerEmail || "") + " | TOTAL: $" + totalFormatted + " | ASISTENTES: " + asistentes.join("; "));
    const qrUrl = "https://quickchart.io/qr?text=" + qrPayload + "&size=140&margin=1";
    const qrRes = UrlFetchApp.fetch(qrUrl, { muteHttpExceptions: true });
    if (qrRes.getResponseCode() === 200) {
      const qrB64 = Utilities.base64Encode(qrRes.getContent());
      qrImgTag = '<img src="data:image/png;base64,' + qrB64 + '" style="width:110px; height:110px; border:1px solid #e6d7f0; border-radius:4px; display:inline-block;" alt="QR" />';
    }
  } catch (e) {
    console.warn("No se pudo descargar QR para PDF:", e);
  }

  // Lista de asistentes numerada
  const listaAsistentesHtml = asistentes.map(function(n, i) {
    return '<div style="padding: 4px 0; font-size: 11px; color: #281e32; font-family: Arial, Helvetica, sans-serif;"><span style="color: #ff8c5a; font-weight: bold; margin-right: 6px;">' + (i + 1) + '.</span> ' + n + '</div>';
  }).join("");

  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <style>
        @page {
          size: A4 portrait;
          margin: 14mm 16mm 14mm 16mm;
        }
        body {
          font-family: Arial, Helvetica, sans-serif;
          margin: 0;
          padding: 0;
          color: #1a0f1e;
          background-color: #ffffff;
          -webkit-font-smoothing: antialiased;
        }
      </style>
    </head>
    <body style="font-family: Arial, Helvetica, sans-serif; margin: 0; padding: 0; color: #1a0f1e; background-color: #ffffff;">

      <!-- 1. CABECERA BRANDING LUMINATE (Tabla con fondo oscuro estricto bgcolor="#1a0f1e") -->
      <table width="100%" bgcolor="#1a0f1e" cellpadding="0" cellspacing="0" border="0" style="width: 100%; background-color: #1a0f1e; border-collapse: collapse; border-radius: 4px 4px 0 0;">
        <tr>
          <td bgcolor="#1a0f1e" style="background-color: #1a0f1e; padding: 22px 24px; vertical-align: middle;">
            <div style="color: #ffffff; font-size: 22px; font-weight: bold; letter-spacing: 1.5px; line-height: 1.1; font-family: Arial, Helvetica, sans-serif;">EVENTO LUMINATE</div>
            <div style="color: #ff8c5a; font-size: 10px; font-weight: bold; letter-spacing: 1px; margin-top: 6px; text-transform: uppercase; font-family: Arial, Helvetica, sans-serif;">COMPROBANTE OFICIAL DE ENTRADA</div>
          </td>
          <td bgcolor="#1a0f1e" align="right" style="background-color: #1a0f1e; padding: 22px 24px; text-align: right; vertical-align: middle;">
            <div style="color: #f5c842; font-size: 10px; font-weight: bold; letter-spacing: 1px; text-transform: uppercase; font-family: Arial, Helvetica, sans-serif;">ACREDITACIÓN DIGITAL VÁLIDA</div>
            <div style="color: #c8bed7; font-size: 9.5px; margin-top: 4px; font-family: Arial, Helvetica, sans-serif;">Iglesia Gran Rey • Venta Oficial</div>
          </td>
        </tr>
      </table>

      <!-- 2. RAYAS DE ACENTO DE PALETA (Melocotón 50%, Dorado 25%, Lavanda 25%) -->
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="width: 100%; height: 5px; margin-bottom: 22px; line-height: 5px; font-size: 1px; border-collapse: collapse;">
        <tr>
          <td bgcolor="#ff8c5a" width="50%" height="5" style="background-color: #ff8c5a; height: 5px; font-size: 1px; line-height: 1px;">&nbsp;</td>
          <td bgcolor="#f5c842" width="25%" height="5" style="background-color: #f5c842; height: 5px; font-size: 1px; line-height: 1px;">&nbsp;</td>
          <td bgcolor="#b07aff" width="25%" height="5" style="background-color: #b07aff; height: 5px; font-size: 1px; line-height: 1px;">&nbsp;</td>
        </tr>
      </table>

      <!-- 3. BLOQUES DE INFORMACIÓN (COMPRADOR & COMPROBANTE) -->
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="width: 100%; border-collapse: separate; margin-bottom: 22px;">
        <tr>
          <!-- Tarjeta Izquierda: Datos del Comprador -->
          <td width="48%" bgcolor="#faf6fc" style="background-color: #faf6fc; border: 1px solid #e6d7f0; border-radius: 6px; padding: 14px 16px; vertical-align: top;">
            <div style="color: #ff8c5a; font-size: 10.5px; font-weight: bold; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 10px; font-family: Arial, Helvetica, sans-serif;">DATOS DEL COMPRADOR (TITULAR)</div>
            <div style="font-size: 11px; color: #32283c; line-height: 1.65; font-family: Arial, Helvetica, sans-serif;">
              <div><strong>Email:</strong> ${buyerEmail || '-'}</div>
              <div><strong>WhatsApp:</strong> ${buyerPhone || '-'}</div>
              <div><strong>Iglesia / Ciudad:</strong> ${ciudad || '-'}</div>
              <div><strong>Pastor:</strong> ${pastor || '-'}</div>
            </div>
          </td>
          <!-- Separador -->
          <td width="4%">&nbsp;</td>
          <!-- Tarjeta Derecha: Detalles del Comprobante -->
          <td width="48%" bgcolor="#faf6fc" style="background-color: #faf6fc; border: 1px solid #e6d7f0; border-radius: 6px; padding: 14px 16px; vertical-align: top;">
            <div style="color: #ff8c5a; font-size: 10.5px; font-weight: bold; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 10px; font-family: Arial, Helvetica, sans-serif;">DETALLES DE LA TRANSACCIÓN</div>
            <div style="font-size: 11px; color: #32283c; line-height: 1.65; font-family: Arial, Helvetica, sans-serif;">
              <div><strong>N° de Comprobante:</strong> ${ref}</div>
              <div><strong>Fecha de emisión:</strong> ${fechaStr}</div>
              <div><strong>Hora de emisión:</strong> ${horaStr} hs</div>
              <div style="margin-top: 4px;">
                <strong style="color: #10b981; font-size: 11px; letter-spacing: 0.5px;">Estado: PAGO ACREDITADO ✓</strong>
              </div>
            </div>
          </td>
        </tr>
      </table>

      <!-- 4. TABLA DE ENTRADAS ADQUIRIDAS -->
      <div style="font-size: 11px; font-weight: bold; color: #1a0f1e; text-transform: uppercase; margin-bottom: 8px; letter-spacing: 0.5px; font-family: Arial, Helvetica, sans-serif;">DETALLE DE ENTRADAS ADQUIRIDAS</div>
      <table width="100%" cellpadding="8" cellspacing="0" border="0" style="width: 100%; border-collapse: collapse; margin-bottom: 22px;">
        <thead>
          <tr>
            <th bgcolor="#1a0f1e" align="left" style="background-color: #1a0f1e; color: #ffffff; padding: 10px 12px; font-size: 10px; font-weight: bold; text-align: left; font-family: Arial, Helvetica, sans-serif;">Descripción de Entrada</th>
            <th bgcolor="#1a0f1e" align="center" width="50" style="background-color: #1a0f1e; color: #ffffff; padding: 10px 12px; font-size: 10px; font-weight: bold; text-align: center; font-family: Arial, Helvetica, sans-serif;">Cant.</th>
            <th bgcolor="#1a0f1e" align="right" width="110" style="background-color: #1a0f1e; color: #ffffff; padding: 10px 12px; font-size: 10px; font-weight: bold; text-align: right; font-family: Arial, Helvetica, sans-serif;">Precio Unitario</th>
            <th bgcolor="#1a0f1e" align="right" width="110" style="background-color: #1a0f1e; color: #ffffff; padding: 10px 12px; font-size: 10px; font-weight: bold; text-align: right; font-family: Arial, Helvetica, sans-serif;">Subtotal</th>
          </tr>
        </thead>
        <tbody>
          <tr bgcolor="#faf8fc" style="background-color: #faf8fc; color: #32283c; border-bottom: 1px solid #ebdcf5;">
            <td style="padding: 10px 12px; font-size: 11px; font-family: Arial, Helvetica, sans-serif;">Entrada Individual / General (Acceso Oficial)</td>
            <td align="center" style="padding: 10px 12px; font-size: 11px; text-align: center; font-family: Arial, Helvetica, sans-serif;">${cant}</td>
            <td align="right" style="padding: 10px 12px; font-size: 11px; text-align: right; font-family: Arial, Helvetica, sans-serif;">$${precioUnitFormatted}</td>
            <td align="right" style="padding: 10px 12px; font-size: 11px; text-align: right; font-family: Arial, Helvetica, sans-serif;">$${totalFormatted}</td>
          </tr>
          <tr bgcolor="#fff8f0" style="background-color: #fff8f0;">
            <td colspan="3" align="right" bgcolor="#fff8f0" style="background-color: #fff8f0; text-align: right; font-weight: bold; color: #ff8c5a; font-size: 11px; padding: 10px 12px; text-transform: uppercase; border: 1px solid #ff8c5a; border-right: none; font-family: Arial, Helvetica, sans-serif;">
              TOTAL OFICIAL PAGADO:
            </td>
            <td align="right" bgcolor="#fff8f0" style="background-color: #fff8f0; text-align: right; font-weight: bold; color: #1a0f1e; font-size: 13px; padding: 10px 12px; border: 1px solid #ff8c5a; border-left: none; font-family: Arial, Helvetica, sans-serif;">
              $${totalFormatted}
            </td>
          </tr>
        </tbody>
      </table>

      <!-- 5. LISTA OFICIAL DE ASISTENTES REGISTRADOS -->
      <div style="font-size: 11px; font-weight: bold; color: #1a0f1e; text-transform: uppercase; margin-bottom: 8px; letter-spacing: 0.5px; font-family: Arial, Helvetica, sans-serif;">LISTA OFICIAL DE ASISTENTES REGISTRADOS (${asistentes.length} Personas)</div>
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="width: 100%; margin-bottom: 24px; border-collapse: separate;">
        <tr>
          <td bgcolor="#faf8fc" style="background-color: #faf8fc; border: 1px solid #e6d7f0; border-radius: 6px; padding: 14px 16px;">
            ${listaAsistentesHtml}
          </td>
        </tr>
      </table>

      <!-- 6. PIE DE PÁGINA CON QR Y REGLAS DE ACREDITACIÓN -->
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="width: 100%; border-top: 1px solid #dcd2e6; padding-top: 14px; border-collapse: collapse;">
        <tr>
          <td style="vertical-align: top; padding-right: 15px; font-family: Arial, Helvetica, sans-serif;">
            <div style="font-size: 10px; font-weight: bold; color: #1a0f1e; margin-bottom: 6px; text-transform: uppercase;">
              INFORMACIÓN IMPORTANTE PARA LA ACREDITACIÓN:
            </div>
            <div style="font-size: 9px; color: #5a5064; line-height: 1.55;">
              <div>• Presentar este comprobante oficial en formato digital (celular) o impreso.</div>
              <div>• Cada asistente debe concurrir con su DNI para la entrega de la pulsera oficial.</div>
              <div>• Entrada intransferible sin previa autorización de los organizadores.</div>
            </div>
            <div style="font-size: 9.5px; font-weight: bold; color: #ff8c5a; margin-top: 8px;">
              Soporte oficial WhatsApp: +54 9 336 4333287 • Iglesia Gran Rey
            </div>
            <div style="font-size: 8px; color: #8c8296; font-style: italic; margin-top: 4px;">
              Documento digital generado automáticamente y verificado por el sistema de Luminate.
            </div>
          </td>
          <td width="115" align="right" style="width: 115px; text-align: right; vertical-align: middle;">
            ${qrImgTag}
          </td>
        </tr>
      </table>
    </body>
    </html>
  `;
}

// --- ENVÍO DE COMPROBANTE OFICIAL POR CORREO (CON COPIA A FACTURACIÓN Y PDF ADJUNTO UNIFICADO) ---
function enviarComprobantePorEmail(destinatario, telefono, ref, paymentId, asistentes, totalMonto, ciudad, pastor) {
  try {
    const asunto = "🎟️ Tu Comprobante y Entradas Oficiales - EVENTO LUMINATE";
    const listaHtml = asistentes.map(function(n, idx) {
      return "<li style='padding:6px 0; border-bottom:1px solid rgba(255,140,90,0.2);'><strong>" + (idx + 1) + ".</strong> " + n + " <span style='color:#ff8c5a;'>(Entrada Confirmada)</span></li>";
    }).join("");

    const totalFormatted = Number(totalMonto).toLocaleString('es-AR');

    const htmlBody = `
      <div style="font-family: Arial, sans-serif; max-width:600px; margin:0 auto; background:#1a0f1e; color:#fff5ee; border-radius:16px; overflow:hidden; border:1px solid #ff8c5a;">
        <div style="background: linear-gradient(135deg, #2b1335, #1a0f1e); padding:32px 24px; text-align:center; border-bottom:2px solid #ff8c5a;">
          <h1 style="color:#ff8c5a; margin:0; font-size:26px; letter-spacing:2px;">✨ EVENTO LUMINATE</h1>
          <p style="color:#f5c842; font-weight:bold; margin-top:8px; font-size:14px; text-transform:uppercase;">¡Pago Confirmado y Registro Exitoso!</p>
        </div>
        <div style="padding:28px 24px;">
          <p style="font-size:16px; line-height:1.6;">Hola, confirmamos que tu pago se acreditó correctamente en nuestro sistema. Ya están aseguradas tus entradas oficiales para <strong>EVENTO LUMINATE</strong>.</p>
          
          <div style="background:rgba(255,255,255,0.06); border-radius:12px; padding:18px; margin:20px 0; border:1px solid rgba(255,140,90,0.3);">
            <p style="margin:4px 0;"><strong>📋 Referencia:</strong> ${ref}</p>
            <p style="margin:4px 0;"><strong>💳 ID Transacción MP:</strong> ${paymentId}</p>
            <p style="margin:4px 0;"><strong>📍 Congregación / Ciudad:</strong> ${ciudad || '-'}</p>
            <p style="margin:4px 0;"><strong>👤 Pastor:</strong> ${pastor || '-'}</p>
            <p style="margin:4px 0; font-size:18px; color:#f5c842;"><strong>💰 Total Abonado:</strong> $${totalFormatted}</p>
          </div>

          <h3 style="color:#ff8c5a; margin-top:24px; font-size:16px;">👥 Asistentes acreditados (${asistentes.length}):</h3>
          <ul style="list-style:none; padding:0; margin:12px 0; font-size:14px;">
            ${listaHtml}
          </ul>

          <div style="text-align:center; margin: 30px 0 20px;">
            <a href="https://luminate.iglesiagranrey.com/registro-exitoso.html?ref=${ref}&status=approved" 
               style="background: linear-gradient(135deg, #ff8c5a, #f5c842); color: #1a0f1e; text-decoration: none; padding: 14px 28px; border-radius: 999px; font-weight: bold; font-size: 15px; display: inline-block; box-shadow: 0 4px 15px rgba(255,140,90,0.4);">
              📥 Descargar Comprobante (PDF) / Ver Pulsera Digital
            </a>
            <p style="font-size:12px; color:#c4a88a; margin-top:10px;">
              📎 Te adjuntamos tu comprobante oficial en formato PDF idéntico al de la web con tu código QR de acceso.
            </p>
          </div>

          <div style="background:#24142a; padding:16px; border-radius:10px; margin-top:24px; border-left:4px solid #f5c842;">
            <p style="margin:0; font-size:13px; color:#c4a88a;">
              ℹ️ <strong>Información importante:</strong> En la mesa de entrada del evento, cada asistente debe presentarse con su DNI para el retiro de su pulsera oficial.
            </p>
          </div>
        </div>
        <div style="background:#120a15; padding:16px; text-align:center; font-size:12px; color:#c4a88a;">
          <p style="margin:0; font-weight:bold; color:#f5c842;">Iglesia Gran Rey · Evento Luminate</p>
          <p style="margin:4px 0 0;">Comprobante de operación emitido automáticamente por el sistema de eventos.</p>
          <p style="margin:4px 0 0; color:#ff8c5a;">Contacto administración: ${EMAIL_FACTURACION}</p>
        </div>
      </div>
    `;

    // 📎 Generar PDF oficial idéntico al de la web con código QR
    let pdfBlob = null;
    let pdfBase64 = null;
    try {
      const htmlPdf = generarHtmlComprobantePdf(ref, paymentId, asistentes, totalMonto, ciudad, pastor, destinatario, telefono);
      pdfBlob = Utilities.newBlob(htmlPdf, "text/html", "Comprobante_Luminate_" + ref + ".pdf").getAs("application/pdf");
      pdfBase64 = Utilities.base64Encode(pdfBlob.getBytes());
    } catch (pdfErr) {
      console.warn("Aviso: No se pudo generar PDF adjunto, se enviará el correo HTML:", pdfErr);
    }

    const resendApiKey = getResendApiKey();
    let envioExitoso = false;

    // 1. DESPACHO OFICIAL POR RESEND API (facturacion@entradas.iglesiagranrey.com)
    if (resendApiKey && resendApiKey.trim() !== "" && destinatario && String(destinatario).includes("@")) {
      try {
        const payloadResend = {
          from: NOMBRE_REMITENTE + " <" + EMAIL_FACTURACION + ">",
          to: [String(destinatario).trim()],
          subject: asunto,
          html: htmlBody,
          reply_to: EMAIL_RESPUESTAS
        };

        if (pdfBase64) {
          payloadResend.attachments = [
            {
              filename: "Comprobante_Luminate_" + ref + ".pdf",
              content: pdfBase64
            }
          ];
        }

        if (!destinatario.toLowerCase().includes(EMAIL_RESPUESTAS.toLowerCase())) {
          payloadResend.bcc = [EMAIL_RESPUESTAS];
        }

        const res = UrlFetchApp.fetch("https://api.resend.com/emails", {
          method: "post",
          contentType: "application/json",
          headers: {
            Authorization: "Bearer " + resendApiKey.trim()
          },
          payload: JSON.stringify(payloadResend),
          muteHttpExceptions: true
        });

        const resCode = res.getResponseCode();
        const resText = res.getContentText();
        console.log("Respuesta Resend (" + resCode + "): " + resText);

        if (resCode === 200 || resCode === 201) {
          envioExitoso = true;
          console.log("Comprobante oficial con PDF unificado despachado por Resend a: " + destinatario);
        } else {
          console.warn("Resend respondió HTTP " + resCode + ". Se activará fallback GmailApp.");
        }
      } catch (resendErr) {
        console.error("Error al despachar por Resend API:", resendErr);
      }
    }

    // 2. FALLBACK SEGURO POR GMAILAPP:
    if (!envioExitoso) {
      console.log("Despachando comprobante mediante GmailApp (Fallback)...");
      const targetMail = (destinatario && String(destinatario).includes("@")) ? destinatario : EMAIL_RESPUESTAS;
      const mailOptions = {
        htmlBody: htmlBody,
        name: NOMBRE_REMITENTE,
        replyTo: EMAIL_RESPUESTAS
      };
      if (pdfBlob) {
        mailOptions.attachments = [pdfBlob];
      }
      GmailApp.sendEmail(targetMail, asunto, "Tu pago para EVENTO LUMINATE fue confirmado con éxito. Referencia: " + ref, mailOptions);
      console.log("Comprobante enviado por GmailApp a: " + targetMail);
    }

  } catch (mailErr) {
    console.warn("No se pudo enviar el correo de comprobante:", mailErr);
  }
}
