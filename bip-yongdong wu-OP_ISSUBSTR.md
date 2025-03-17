<pre>
  BIP: 
  Title: OP_ISSUBSTR
  Author: Yongdong Wu <wuyd007@qq.com>
          Weichu Deng <weichudeng@stu2024.jnu.edu.cn>
          Jian Weng <cryptjweng@gmail.com>
  Comments-URI: 
  Status: draft
  Type: Standards Track
  Created: 2025-03-17
  License: BSD-3-Clause
</pre>



## Abstract

This BIP introduces an opcode for strings, `OP_ISSUBSTR` and `OP_ISSUBSTRVERIFY`(similar to the relationship between `OP_EUQUAL` and `OP_EUQUALVERIFY`), which determines whether a string is a substring of another string. As this opcode does not change any status of the blockchain, it is secure.

## Specification

This opcode checks if the second string on the stack is a substring of the first string. If the opcode is `OP_ISSUBSTRVERIFY`, it also verifies the condition and throws an error if it is false and the result is not retained.  

Execution process:

1. Take the two strings at the top of the stack.
2. Use standard library functions to compare the two strings.
3. The two strings pop off the stack and push the result into the stack.
4. If the opcode is `OP_ISSUBSTRVERIFY`, the result does not push into the stack.

## Motivation

The lack of string operations in Bitcoin scripts limits Bitcoin's application. When developers need to use string operations to build applications, they need to simulate these functions through off-chain preprocessing or complex scripts, which increases the development difficulty and may introduce centralized dependencies.

Early versions of Bitcoin support some string operations, such as `OP_SUBSTR`. This operation extracts a substring of a specified position and length from a string and replaces the original string with the substring. For security reasons, `OP_SUBSTR` is disabled in Bitcoin `v0.3.10` and its subsequent versions <sup>[1]</sup>. The reason for the disablement is a vulnerability accident (`CVE-2010-5137`<sup>[2,3]</sup>), which is caused by `OP_LSHIFT`. To avoid similar overflow vulnerabilities, Bitcoin has disabled a batch of opcodes <sup>[4]</sup>, including `OP_SUBSTR`. With the widespread adoption of Bitcoin, the limitations of lacking string operations have become more apparent. The `OP_ISSUBSTR` we proposed adds string search functions to Bitcoin scripts. This operation does not change any state, so the proposed opcode is safe.

We list the advantages of `OP_ISSUBSTR` below:

### Advantages

1. **Enhanced script functionality and flexibility**
Developers can process string-related logic directly on the chain without relying on off-chain processing. For example: In a multi-signature wallet, developers may need to verify whether certain transactions contain specific signer information or remarks. With `OP_ISSUBSTR`, you can directly check in the script whether the transaction comment or signature field contains a specific substring.
2. **Support string searching**
In some application scenarios, developers may need to verify whether certain parts of a string conform to a specific format or contain specific data. For example, check the payee name in the payment transaction against a pre-set value.
3. **Convert non-deterministic algorithms to deterministic ones**
Some signature algorithms or hash functions may produce non-deterministic outputs. Through `OP_ISSUBSTR`, developers can check whether the output contains certain known substrings in the script, thereby converting the output of non-deterministic algorithms into deterministic results. For example, verify whether the hash value contains a specific hexadecimal sequence (such as `0000`) to trigger a certain logic of the contract.
4. **Simplify address verification logic**
Bitcoin addresses usually start with a specific prefix or suffix. Through `OP_ISSUBSTR`, developers can directly verify whether the address conforms to the expected format in the script. For example, verify whether the transaction target address starts with `bc1` to ensure that the transaction target is a valid Bitcoin address, or detect/defeat "address pollution" attacks.
5. **Integrate with modern programming languages**
Modern programming languages widely support string operations. The introduction of `OP_ISSUBSTR` makes Bitcoin scripts more aligned with these languages, lowering the barrier to entry for developers.


## Reference Implementation

```C++
case OP_ISSUBSTR:
case OP_ISSUBSTRVERIFY: {
    // Check whether the stack has at least two elements 
    if (stack.size() < 2)
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);

    // Get the two strings at the top of the stack
    valtype& vch1 = stacktop(-2); // The checked string 
    valtype& vch2 = stacktop(-1); // substring

    // Check whether vch2 is a substring of vch1
    bool is_substr = std::search(vch1.begin(), vch1.end(), vch2.begin(), vch2.end()) != vch1.end();

    // Populate two elements at the top of the stack
    popstack(stack);
    popstack(stack);

    // Push the result (true or false) back on the stack
    stack.push_back(is_substr ? vchTrue : vchFalse);
    if (opcode == OP_ISSUBSTRVERIFY)
        {
            if (is_substr)
                popstack(stack);
            else
                return set_error(serror, SCRIPT_ERR_EQUALVERIFY);
        }
    }
break;
```

## References

1. [Blaming bitcoin/script.cpp at v0.3.10 · bitcoin/bitcoin](https://github.com/bitcoin/bitcoin/blame/v0.3.10/script.cpp)
1. [NVD - CVE-2010-5137](https://nvd.nist.gov/vuln/detail/CVE-2010-5137)
1. [Common Vulnerabilities and Exposures - Bitcoin Wiki](https://en.bitcoin.it/wiki/Common_Vulnerabilities_and_Exposures#CVE-2010-5137)
1. [misc changes · bitcoin/bitcoin@4bd188c](https://github.com/bitcoin/bitcoin/commit/4bd188c4383d6e614e18f79dc337fbabe8464c82)

## Copyright

This document is licensed under the 3-clause BSD license.
