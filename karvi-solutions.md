# Karvi-geddon: How a Restaurant Ordering Platform Became a Security Catastrophe

Welcome back, dear reader! Today we're serving up something special from the world of catastrophic security failures. Grab your popcorn (or maybe don't order it through Karvi Solutions), because we're about to dive into one of the most spectacular security dumpster fires we've seen in the restaurant tech space.

Our journey into this security nightmare began after Heise Online and the Chaos Computer Club already exposed Karvi Solutions' appalling security practices. The CCC published a devastating report showing how anyone could access customer orders and, even more incredibly, download the company's entire source code directly from the web root using a simple `wget` command. Yes, you read that right - their comPlete source code was just sitting there, exposed for anyone to grab.

Inspired by their findings (and frankly horrified by the implications), we managed to get our hands on a copy of the application and dove into their source code. What we discovered wasn't just a vulnerability or two. Oh no, that would be too simple. We found a complete security apocalypse that makes you wonder if the developers were actively trying to create the most insecure payment system imaginable.

**The short version?** Every single customer credit card stored in this system is effectively public. The encryption is a joke, the SQL injection is everywhere, and the company's response to our security disclosures was... complete silence.

Let's dig in, shall we?

## The Encryption

First up, let's talk about what Karvi Solutions calls "encryption." We put it in quotes because calling this encRyption is like calling a paper bag a safe.

The system claims to protect customer credit card data with some fancy cryptographic operations. Here's their "secure" obfuscation:

```php
define('SEED', 'Food2Door');

function ccEncrypt($password, $time){
    //appending padding characters
    $newPass = rand(0,9) . rand(0,9);
    $c = 1;
    while ($c < 15 && (int)substr($newPass,$c-1,1) + 1 != (int)substr($newPass,$c,1)){
        $newPass .= rand(0,9);
        $c++;
    }
    $newPass .= $password;

    //applying XOR
    $newSeed = md5(SEED . $time);
    $passLength = strlen($newPass);
    while (strlen($newSeed) < $passLength) $newSeed.= $newSeed;
    $result = (substr($newPass,0,$passLength) ^ substr($newSeed,0,$passLength));

    return base64_encode($result);
}
```

Oh, where to even begin with this masterpiece of security incompetence?

Let's start with the hardcoded seed: `'Food2Door'`. That's right, every single installation of this software uses the exact same "secret" key. It's like every house in the neighborhood using the same lock and leaving the key under the same doormat.

But wait, it gets better! The obfuscation uses XOR with a key derived from this hardcoded seed plus a timestamp. And where is that timestamp stored? You guessed it - right in the database next to the "obfuscated" credit card number.

This means anyone with database access can reverse the obfuscation and recover every single credit card. There is literally no protection here - just the illusion of security.

And the cherry on top? They're storIng CVV codes. Yes, the 3-4 digit codes that PCI DSS explicitly forbids from being stored. They're not just breaking security best practices; they're violating payment card industry rules in the most spectacular way possible.

Want to see exactly what they're storing? When we dove into their database structure, we found the `rt_credit_cards` table containing everything an attacker would need:

- `cc_number` (varchar) - the full 16-digit credit card number
- `cc_cvv` (varchar) - the 3-4 digit CVV code that PCI DSS explicitly forbids from being stored
- `cc_exp_month` and `cc_exp_year` - the complete expiration date
- `cc_time` - the timestamp used in their "encryption" key generation
- `customer_id` - linking it all to specific customers

Yes, you read that right. They're storing full credit card numbers AND CVV codes together in their database, "protected" by their joke encryption. Every single piece of data needed to use these credit cards is sitting there, waiting to be recovered by anyone with database access - or, you know, anyone who finds one of the numerous SQL injection vulnerabilities scattered throughout their codebase.

## SQL Injection: The Gift That Keeps on Giving

If you thought the "encryption" was bad, wait until you see how they handle database queries.

Here's a taste of their database security:

```php
function getorderupdateapi() {
    if(!empty($_POST['orderid'])){
        $od = $_POST['orderid'];
        $odd = $this->db->query("SELECT * FROM `rt_session_tmp` where orderid='{$od}'")->row();
```

Yes, that's direct string concatenation of user input into SQL queries. No parameterization, no sanitization, just raw vulnerability waiting to be exploited.

And this isn't an isolated incident. SQL injection vulnerabilities are present throughout the application.
We found them in order management, customer data handling, administrative functions - basically anywhere the application touches the database without proper parameterization.

**Security Disclosure Notice:** We're intentionally omitting the exact HTTP requests and payload details from this blog post.
Why? Because publishing step-by-step instructioNs for exploiting these vulnerabilities would be like handing a loaded gun to every script kiddie on the internet.
The vulnerabilities are trivial to find and exploit, but we refuse to be the ones who help criminals destroy hundreds of small businesses that rely on this platform.

## The Company That Couldn't Care Less

Here's where the story gets really interesTing (and depressing). After discovering these catastrophic vulnerabilities, we did what responsible security researchers do: we tried to contact Karvi Solutions.

We sent detailed vulnerability reports. We waited.

**Complete silence.**

This is particularly concerning given Karvi Solutions' documented history of security failures. The Chaos Computer Club previously discovered that Karvi Solutions left their entire source code exposed as a ZIP file in the web root. Heise Online has reported on previous security issues with their software.

And now this. A platform handling hundreds of thousands of orders with security so bad it's almost impressive in its incompetence.

## The Impact: A Security Apocalypse

Let's talk about what's actually at risk here. This isn't somE theoretical vulnerability that might be exploited under specific conditions. This is a ticking time bomb with:

- **Full credit card numbers** (all 16 digits) for every customer
- **CVV codes** (explicitly forbidden by PCI DSS)
- **Customer names, addresses, phone numbers**
- **Complete order histories**
- **Restaurant business data**
- **Administrative credentials**

Every single restaurant using this platform is at risk. Every single customer who has ordered through these systems has had their payment data compromised by design.

## Beyond SQL Injection: The Language Files That Speak Hacker

But wait, it gets worse. As if the "encryption" disaster and SQL injection apocalypse weren't enough, we discovered something that takes this from "catastrophic" to "unbelievable."

The Karvi platform includes language management functionality that allows updating translation files. Sounds innocent enough, right? Wrong. This feature is a security nightmare that makes everything else look like child's play.

### The Vulnerable Code

The language update controllers (`cilanguage.php` and `cilanguage_superadmin.php`) contain some of the most dangerous code:

```php
foreach($this->input->post() as $key => $values) {
    if( $key != "fakeusernameremembered" && $key != "fakeuseremailremembered" && $key != "fakepasswordremembered" && $key != "restaurant_id" && $key != "languagefor" ) {
        $arrayKeys .= '$lang["'.$key.'"]'.str_repeat(' ', 4).'= "'.$values.'";'.PHP_EOL;
    }
}
$fp = fopen(ROOT_PATH.VENDOR_CI_LANG_FILE.$languagefor.'/'.$fileName, 'w');
$fileStatus = fwrite($fp, '<?php'.PHP_EOL.$arrayKeys.'?>');
```

Let's break down why this is so terrifying:

1. **No Authentication Required** - Don get fooled by `superadmin`.
2. **No Input Sanitization** - User input goes directly into PHP code without validation
3. **Arbitrary File Write** - Creates PHP files in locations controlled by the `languagefor` parameter
4. **Path Traversal** - The `languagefor` parameter allows directory escape
5. **Immediate Execution** - Created files are web-accessible and executable

### The Attack Vector

Anyone can send a POST request to the language update endpoint with malicious parameters. No login required, no credentials needed. The system happily takes user input, wraps it in PHP syntax, and writes it to a file that gets executed by the web server.

The path traversal vulnerability allows escaping protected directories, meaning an attacker can write malicious PHP files anywhere on the server - including web-accessible directories where they'll be immediately executed.

### The Systemic Security Failure

What makes this particularly horRifying is that it exists in **both** the restaurant admin and superadmin controllers. This isn't an oversight - it's a fundamental failure to understand basic security principles.

The fact that code this dangerous made it into production shows a complete lack of security review, testing, or basic competence. This isn't just bad coding - it's serious negligence.

## And It Gets Worse: Order Receipts in the Web Root

But wait, there's more! As if the RCE vulnerability wasn't enough, we discovered yet another security disaster that exposes customer data directly to the internet.

The main platform (netuptakeaway.de) stores order receipts as plain text files in the web root with predictable filenames like `[REST_ID]receipt.txt`. Anyone who can guess an order ID can access complete order details from ANY restaurant using the platform, including:

- Customer names and contact information
- Complete order contents and prices
- Delivery addresses
- Payment information
- Order timestamps

The files are stored in publicly accessible directories on the main platform like `/P[REDACTED]/OrderFiles/` with URLs that are easily guessable and indexable by search engines. This means customer order data from ALL restaurants using the platform is essentially public domain - no authentication required, no hacking needed, just simple URL enumeration of the main platform.

This isn't just a vulnerability; it's a complete failure to understand basic web security principles. Sensitive customer data should never be stored in web-accessible directories, especially with predictable filenames.

## The Shared Host Nightmare

We've been asked to confirm claims circulating that around 350 restaurants are affected by this security catastrophe. **Let us be crystal clear: this narrative is fundamentally misleading.**

Here's what our analysis reveals: the "350" number refers to restaurants on one specific shared host where hundreds of restaurants all use the **exact same database with identical credentials**. A single database breach gives attackers access to every restaurant's data on that particular host.

But the nightmare doesn't stop there. We've identified other shared hosts with the same architecture - multiple hosting environments each running vulnerable versions of the application. Some hosts run slightly "newer" versions that were clearly "improved" using AI, evidenced by cringe-inducing comments like `// ✅ <something only an AI would comment>`.

The result? Predictably catastrophic. The AI-"enhanced" version introduces **brand new SQL injection vulnerabilities** on top of the existing ones. So we're not looking at one compromised host - we're looking at multiple shared hosts, each with identical fundamental vulnerabilities, some with additional AI-generated security holes.

## Thoughts

- "Order IDs tell stories in the AI based review systems - sometimes more than intended."

## What Should Happen Now

**The Karvi Solutions restaurant ordering platform must be immediately shut down.**

This isn't a vulnerability that needs patching. This is a fundamental architectural failure that requires complete redevelopment. The current state of this software makes it gross negligence to continue operating it, especially given the sensitive nature of payment data.

Restaurant owners using this platform should:

1. **Immediately stop accepting online orders** through this system
2. **Contact their payment processors** about potential PCI DSS violations
3. **Migrate to a secure platform** immediately

## The Bigger Picture

This case highlights a serious problem in the software industry, particularly in the B2B space where companies sell software to non-technical business owners who have no way to evaluate its security.

Karvi Solutions is selling software to restaurants - small businesses that trust the technology they're paying for. They're not security experts. They shouldn't have to be. But when a company provides software for handling credit card payments, there's a basic expectation of competence.

What we found here goes beyond incompetence. The complete refusal to respond to security disclosures, combined with their documented history of security failures, suggests a company that simply doesn't care about security or the safety of their customers' data.

## Final Thoughts

The Karvi Solutions restaurant ordering platform represents everything that's wrong with security in the software industry. It's a perfect storm of:

- Fundamentally broken cryptography
- Widespread injection vulnerabilities
- Complete disregard for industry standards
- Corporate indifference to security

Every day this platform continues operating, more customers are put at risk. Every restaurant that uses it is potentially violating PCI DSS and exposing themselves to massive legal and financial liability.
