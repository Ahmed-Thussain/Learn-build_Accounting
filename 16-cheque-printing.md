# Phase 16 — Cheque Printing

**Tags:** `[Backend]` `[Frontend]`
**Depends on:** Phase 8 (payments)
**You'll finish this phase with:** the ability to print a payment directly onto a pre-printed
Egyptian bank cheque leaf, with every field landing in the right spot, calibrated per bank.

---

## 1. Learning objectives

- Understand why cheque printing needs **absolute millimeter positioning**, not HTML/CSS flow
  layout — you're printing onto a pre-existing paper form, not designing a page from scratch.
- Build a **per-bank calibration** system, since every Egyptian bank's cheque leaf places its
  fields (date box, amount box, payee line) in different positions.
- Convert a number to **words**, in both Arabic and English — a mandatory feature of any real
  cheque (this is what prevents fraudulent alteration of the numeric amount).

## 2. Why this phase now

Payments (Phase 8) already model `method: 'cheque'`. This phase makes that method actually produce
a printable artifact, not just a database row.

---

## 3. Schema: one row per bank's layout

```php
Schema::create('cheque_templates', function (Blueprint $table) {
    $table->id();
    $table->string('bank_name'); // e.g. "National Bank of Egypt", "CIB", "QNB Alahli"
    // All coordinates in millimeters from the top-left of the physical cheque leaf —
    // measure these against a real blank cheque from that bank with a ruler, once,
    // and they never need to change unless the bank redesigns its cheque.
    $table->decimal('date_x', 6, 2);
    $table->decimal('date_y', 6, 2);
    $table->decimal('payee_x', 6, 2);
    $table->decimal('payee_y', 6, 2);
    $table->decimal('amount_figures_x', 6, 2);
    $table->decimal('amount_figures_y', 6, 2);
    $table->decimal('amount_words_x', 6, 2);
    $table->decimal('amount_words_y', 6, 2);
    $table->decimal('amount_words_line2_x', 6, 2)->nullable(); // long amounts wrap to a 2nd line
    $table->decimal('amount_words_line2_y', 6, 2)->nullable();
    $table->decimal('memo_x', 6, 2)->nullable();
    $table->decimal('memo_y', 6, 2)->nullable();
    $table->decimal('page_width_mm', 6, 2)->default(180); // typical Egyptian cheque leaf width
    $table->decimal('page_height_mm', 6, 2)->default(80);
    $table->timestamps();
});
```

*Why store this per-bank in the database instead of one hardcoded PDF layout:* your users bank
with different banks. A configurable template means adding support for a new bank is a data-entry
task (measure the cheque, fill in the form), not a code change and redeploy.

---

## 4. Amount-to-words — the fraud-prevention feature

A cheque's written-out amount ("One Thousand Two Hundred Fifty Egyptian Pounds Only") is what
actually governs the payment legally if it ever conflicts with the numeral — this is *why* it's
mandatory, not decorative.

```php
class NumberToWordsService
{
    public function toEnglishWords(float $amount): string
    {
        $whole = (int) $amount;
        $piastres = (int) round(($amount - $whole) * 100);

        // Use a maintained library rather than hand-rolling number-to-words —
        // this is a well-solved, easy-to-get-subtly-wrong problem (hundreds vs
        // "one hundred", "and" placement, etc.).
        $words = (new \NumberFormatter('en', \NumberFormatter::SPELLOUT))->format($whole);
        $result = ucfirst($words) . ' Egyptian Pounds';
        if ($piastres > 0) {
            $result .= ' and ' . (new \NumberFormatter('en', \NumberFormatter::SPELLOUT))->format($piastres) . ' Piastres';
        }
        return $result . ' Only';
    }

    public function toArabicWords(float $amount): string
    {
        // PHP's intl NumberFormatter also supports Arabic spellout via the 'ar' locale.
        // Arabic amount-in-words grammar (gender agreement, hundreds/thousands forms)
        // has real edge cases NumberFormatter doesn't fully cover for currency phrasing —
        // for production use, verify output against a native Arabic speaker's review,
        // or a dedicated Arabic-number-to-words package, before this goes live.
        $whole = (int) $amount;
        $words = (new \NumberFormatter('ar', \NumberFormatter::SPELLOUT))->format($whole);
        return "$words جنيه مصري فقط لا غير";
    }
}
```

---

## 5. Generating the PDF, positioned exactly

```powershell
composer require mpdf/mpdf
```

```php
class ChequePrintService
{
    public function __construct(private NumberToWordsService $words) {}

    public function generate(Payment $payment, ChequeTemplate $template): string
    {
        $mpdf = new \Mpdf\Mpdf([
            'format' => [$template->page_width_mm, $template->page_height_mm],
            'margin_left' => 0, 'margin_right' => 0, 'margin_top' => 0, 'margin_bottom' => 0,
        ]);

        // mPDF positions elements with absolute CSS `position: absolute; top/left`
        // measured from the page's top-left corner — this is what "prints onto a
        // pre-printed form correctly" actually requires, unlike normal flowing HTML.
        $html = $this->positionedSpan($template->date_x, $template->date_y, $payment->paid_on->format('d/m/Y'))
              . $this->positionedSpan($template->payee_x, $template->payee_y, $payment->invoice->contact->name)
              . $this->positionedSpan($template->amount_figures_x, $template->amount_figures_y, number_format($payment->amount, 2))
              . $this->positionedSpan($template->amount_words_x, $template->amount_words_y, $this->words->toArabicWords($payment->amount))
              . $this->positionedSpan($template->memo_x, $template->memo_y, "Payment for {$payment->invoice->invoice_number}");

        $mpdf->WriteHTML($html);
        return $mpdf->Output('', 'S'); // return raw PDF bytes
    }

    private function positionedSpan(float $x, float $y, string $text): string
    {
        return "<div style='position:absolute; left:{$x}mm; top:{$y}mm; font-size:11pt;'>{$text}</div>";
    }
}
```

---

## 6. A calibration workflow, not guesswork

Before trusting a bank's coordinates, print a **test grid**:
```php
public function generateCalibrationGrid(ChequeTemplate $template): string
{
    // Draws a 10mm-spaced ruled grid over the exact page size, with mm labels —
    // print this on a real cheque-sized blank sheet, hold it against an actual
    // blank cheque from the bank, and read off the mm coordinates for each field
    // directly, rather than estimating them by eye from a photo.
}
```
Frontend, `frontend/src/views/ChequeTemplatesView.vue`: a form for the coordinate fields, a
"Print Calibration Grid" button, and once positions are set, a "Print Test Cheque" button that
generates the real layout on a blank sheet held up to the light against a real cheque leaf, before
ever printing on an actual cheque with real ink.

---

## 7. What you built, and why it matters later

- A calibration system that turns "get cheque printing right" from trial-and-error with real,
  costly cheque leaves into a repeatable, data-driven setup step per bank.
- The amount-to-words function is reusable wherever a legal/official amount needs spelling out
  (e.g., a formal receipt or a contract-adjacent document later).

## 8. Next: Phase 17

With the core financial features complete, Phase 17 opens the app up to e-commerce platforms —
letting a WooCommerce or Shopify order automatically create an invoice through the same
`InvoiceService` built in Phase 8.
