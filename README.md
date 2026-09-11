# MJ-FacturX

Module Access/VBA qui récupère des factures venant de logiciels spécifiques (Access/Sql/Excel/..) et génère des factures Factur-X (EN16931), CIUS-FR, PDF/A-3, validation Mustang, journal des ventes, intégration PME/artisans — connecté à Chorus Pro (B2G) et aux PDP/Peppol (B2B) pour la réforme de facturation électronique. 

# MJ‑FacturX — Architecture Technique (EN16931 + CIUS‑FR + PDF/A‑3)

MJ‑FacturX est une implémentation Access/VBA de la norme **Factur‑X**, incluant :
- génération XML EN16931 + CIUS‑FR,
- production PDF/A‑3 avec attachement XML,
- validation Mustang,
- pipeline complet de transformation Access → Factur‑X.

---

# 🧩 Architecture globale du système

```
                 +----------------------+
                 |   Application Access |
                 |  (Tables + Reports)  |
                 +----------+-----------+
                            |
                            | invoiceId
                            v
                 +----------------------+
                 |   MJ-FacturX Core    |
                 |  (Modules VBA / src) |
                 +----------+-----------+
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
+---------------+   +----------------+   +------------------+
| XML Generator |   | PDF/A-3 Engine |   | Mustang Validator |
|  EN16931/FR   |   |  Attach XML    |   |  CLI / API        |
+-------+-------+   +--------+-------+   +---------+---------+
        |                    |                     |
        | xmlPath            | pdfPath             | results
        v                    v                     v
+---------------+   +----------------+   +------------------+
| XML Output    |   | PDF/A-3 Output |   | Validation Report |
+---------------+   +----------------+   +------------------+
```

---

# 🧬 Architecture interne (modules)

```
/src
 ├── fx_core.bas        → moteur XML (DOM, mapping, CIUS-FR)
 ├── fx_pdf.bas         → PDF/A-3, XMP, attachement XML
 ├── fx_utils.bas       → formatage, TVA, arrondis, logs
 ├── fx_mustang.bas     → wrapper Mustang (CLI/API)
 └── fx_models.cls      → structures internes (Invoice, Seller, Buyer…)
```

---

# 🧬 Pipeline XML — EN16931 + CIUS‑FR

```
+-------------------+
| Load Invoice Data |
+---------+---------+
          |
          v
+-------------------+
| Map EN16931 Model |
| - Header          |
| - Seller/Buyer    |
| - Lines           |
| - Totals          |
+---------+---------+
          |
          v
+-------------------+
| Build XML DOM     |
| - Namespaces      |
| - CIUS-FR rules   |
| - Monetary totals |
+---------+---------+
          |
          v
+-------------------+
| Save XML File     |
+-------------------+
```

### Exemple VBA interne

```vba
Dim inv As FX_Invoice
Set inv = FX_LoadInvoice(invoiceId)

Call FX_MapHeader(inv)
Call FX_MapSeller(inv)
Call FX_MapBuyer(inv)
Call FX_MapLines(inv)
Call FX_MapTotals(inv)

Call FX_SaveXml(FX_BuildDom(inv), xmlPath)
```

---

# 🖨️ Pipeline PDF/A‑3 — Attachement XML

```
+---------------------------+
| Export Access Report PDF |
+-------------+-------------+
              |
              v
+---------------------------+
| Convert to PDF/A-3        |
| (Ghostscript pipeline)    |
+-------------+-------------+
              |
              v
+---------------------------+
| Attach XML (AFRelationship)|
| Inject XMP Metadata        |
+-------------+-------------+
              |
              v
+---------------------------+
| PDF/A-3 Final Output      |
+---------------------------+
```

### Exemple VBA

```vba
DoCmd.OutputTo acOutputReport, "rptInvoice", acFormatPDF, pdfPath
Call FX_PdfAttachXml(pdfPath, xmlPath)
```

---

# 🧪 Pipeline Mustang — Validation complète

```
+---------------------------+
| Mustang CLI / API         |
+-------------+-------------+
              |
              v
+---------------------------+
| Validate XML              |
| Validate PDF/A-3          |
| Validate Factur-X         |
| Validate CIUS-FR          |
+-------------+-------------+
              |
              v
+---------------------------+
| JSON Report               |
+---------------------------+
```

### Exemple VBA

```vba
result = FX_MustangValidate(xmlPath, pdfPath)

If result.IsValid Then
    Debug.Print "Factur-X OK"
Else
    Debug.Print Join(result.Errors, vbCrLf)
End If
```

---

# 🔌 Intégration Access

```
Access Tables → MJ-FacturX → XML → PDF/A-3 → Mustang → Factur-X Validé
```

Appel principal :

```vba
Call FX_GenerateFacturX(invoiceId)
```

Tables requises :
- `tblSeller`
- `tblBuyer`
- `tblInvoice`
- `tblInvoiceLines`
- `tblVatRates`

---

# 📅 Roadmap technique

- UBL 2.3
- Messages e‑reporting
- Connecteurs API PDP
- Interface Access avancée
- Génération Chorus Pro Qualif

---

# 👤 Auteur

Marie‑Jean Charlier  
Développeur Access/VBA — Factur‑X  
Saint‑Sorlin‑en‑Valloire, France  
charliermariejean@gmail.com

---

# 🔒 Dépôt privé

Toute diffusion ou réutilisation du code nécessite une autorisation préalable.
```
