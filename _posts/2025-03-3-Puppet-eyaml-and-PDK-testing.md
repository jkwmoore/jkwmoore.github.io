---
layout: post
title: "Puppet PDK testing when using hiera eyaml"
category: Puppet
---

> Warning: I cannot claim to be a Puppet backend expert but I managed to get past PDK testing issues as below.

---

### What was the problem?

PDK tests are failing because I don't have the keys necessary to decrypt the secrets while testing locally. 
I also do not want to keep the encryption keys locally, nor anywhere except the PuppetServer.

So what do these errors look like?

Well, I am trying to use the following from my hiera ``common.eyaml``:

```yaml
myclass::mysecretstring: ENC[PKCS7,MYENCRYPTEDSECRETSTRINGCONTENT]
```

And PDK gives me:

```shell
  1) myclass::mymanifest on redhat-8-x86_64 is expected to compile into a catalogue without dependency cycles
     Failure/Error: it { is_expected.to compile }
       error during compilation: Function lookup() did not find a value for the name 'myclass::mysecretstring' on node my_dev_machine.local
     # ./spec/classes/mymanifest_spec.rb:10:in `block (4 levels) in <top (required)>'
```

### How did I sort this?

I added overrides to the spec test used for the class.

```ruby
      before do
        allow(Puppet::Pops::Lookup).to receive(:lookup).and_call_original  # Allow normal lookups

        allow(Puppet::Pops::Lookup).to receive(:lookup)
          .with('myclass::mysecretstring', anything, anything, anything, anything, anything)
          .and_return('MOCKED_VALUE_STRING')
      end
```

### Why do I think this works

Aside of the fact I tested it and it appears to do what I wanted, see below:


#### **What is `Puppet::Pops::Lookup`?**
- Puppet uses **Hiera** to retrieve values from a hierarchy of data sources.
- When a class requests a variable via `lookup('some_variable')`, Puppet internally calls `Puppet::Pops::Lookup.lookup('some_variable', ...)`.
- This function searches for the requested key across the defined Hiera hierarchy.
- https://www.rubydoc.info/gems/puppet/Puppet%2FPops%2FLookup.lookup

For example, if the Puppet manifest has:
```puppet
$mysecret = lookup('myclass::mysecretstring')
```
Puppet will call:
```ruby
Puppet::Pops::Lookup.lookup('myclass::mysecretstring', ...)
```
and return the corresponding value from Hiera.

---

#### Stub Only the Problematic Lookup
```ruby
allow(Puppet::Pops::Lookup).to receive(:lookup)
  .with('myclass::mysecretstring', anything, anything, anything, anything, anything)
  .and_return('MOCKED_VALUE_STRING')
```
- This overrides lookups **only** for `myclass::mysecretstring`.
- `.with(...)` ensures that this stub only activates when the exact expected arguments are passed.
- `.and_return('MOCKED_VALUE_STRING')` means that **whenever Puppet tries to look up this variable, it will get `'MOCKED_VALUE_STRING'` instead of failing.**
- Depending on the expected data type for encrypted values, different mocking values may be required.

**Effect:**  
- `lookup('myclass::mysecretstring')` **returns `'MOCKED_VALUE_STRING'` instead of failing**.  
- All other lookups still use Hiera data lookups as normal/expected.

---

#### Allow Normal Lookups
```ruby
allow(Puppet::Pops::Lookup).to receive(:lookup).and_call_original
```
- `allow(...).to receive(:lookup)` tells RSpec to intercept calls to `Puppet::Pops::Lookup.lookup`.
- `.and_call_original` ensures that all other lookups (except the one we explicitly stub) continue to function normally.
- Without this, RSpec would try to stub all lookups, causing unexpected failures when other values like `myclass::someotherstring'` are requested from ``common.yaml``.

**Effect:**  
- Any `lookup('some_variable')` that is not explicitly stubbed will still perform its normal function.

---

### **tl;dr Why I think this works and warnings**
1. **Ensures normal lookups proceed without modification** → The `.and_call_original` call ensures we do not break unrelated lookups.
2. **Only stubs the problematic keys** → This avoids unnecessary stubbing and allows normal Puppet behavior.
3. **Prevents lookup errors in tests** → PDK tests no longer fail due to "missing" Hiera data.
4. Depending on the expected data type for encrypted values, different mocking values may be required.
5. The ``Method: Puppet::Pops::Lookup.lookup`` method documentation explicitly says:
     `` This method is part of a private API. You should avoid using this method if possible, as it may be removed or be changed in the future. ``
6. I am not a Ruby expert nor Puppet/PDK backend expert - here be dragons!

GLHF!
JKWM.
