# How Create Tests with LLVM Lit Tool

## Lit: LLVM Integrated Tester
- LLVM test runner
- Python Script
- Works outside LLVM Tree -> `pip install lit`

### Reason to Use:
- LLVM based tests
- Test Discovery
- Simple Configuration Files
- Simple Test Scripts

## Using LLVM Lit Tool
- Installing:
    - `pip install lit`
    - LLVM Build

## Creating a Test Suite
Following LLVM Test-Suite structure.

### Example

#### Test Suite Structure:
```sh
my-test-suite/ # Main Directory (root)
 |
 --- lit.cfg   # Lit Configuration File
 |
 --- palindrome  # Program Directory
 |   | 
 |   --- expected_output.txt  # Expected Output "1"
 |   |
 |   --- input.txt  # Input File "revivor"
 |   |
 |   --- palindrome.cpp  # Program Source File
 |   |
 |   --- palindrome.test  # Test File
 |
 --- square
     |
     --- expected_out.txt
     |
     --- input.txt
     |
     --- square.cpp
     |
     --- square.test

```

#### Lit Configuration File (lit.cfg)
```python
import lit.formats

config.name = "My Test Suite Example"
config.test_format = lit.formats.ShTest(True) # Shell Test
config.suffixes = ['.test']
```

#### palindrome.cpp:
```cpp
#include <iostream>
#include <string>

int isPalindrome(std::string word) {
    int palindrome = 1;
    for (int i = 0, j = word.size() - 1; i < j; i++, j--) {
        if (word[i] != word[j]) {
            palindrome = 0;
            break;
        }
    }
    return palindrome;
}

int main() {
    std::string word;
    std::cin >> word;
    std::cout << isPalindrome(word);
    return 0;
}
```

#### palindrome.test:
```llvm
RUN: cd %S; clang++ %S/palindrome.cpp -o %t
RUN: %t < input.txt | diff - expected_output.txt
```

- **Common Macros**:
`- `%S`: Test Directory
- `%t`: Temporary File`

#### Running:
```sh
$ cd my-test-suite

# Entire Test Suite:
$ lit my-test-suite

# Specific Program:
$ lit my-test-suite/palindrome

# Output for Entire Test Suite
$ -- Testing: 2 tests, 2 workers --
$ PASS: My Test Suite Example :: palindrome/palindrome.test (1 of 2)
$ PASS: My Test Suite Example :: square/square.test (2 of 2)

$ Testing Time: 0.58s
$  Passed: 2
```