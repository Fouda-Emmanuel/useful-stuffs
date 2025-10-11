# Checksum Algorithms: Luhn & Modulo 97

A comprehensive guide to two essential checksum algorithms used in financial systems worldwide.

## 🎯 Quick Summary

| Algorithm | Used For | Check Digits | Simple Rule |
|-----------|----------|--------------|-------------|
| **Luhn** | Credit Cards | 1 digit | Sum must be 0 (Total Sum % 10 == 0) |
| **Modulo 97** | Bank Accounts | 2 digits | Remainder must be 1 (Giant Number % 97 == 1)|

---

## 🔢 Luhn Algorithm

### Overview
The **Luhn algorithm** (also known as "modulus 10" algorithm) is a simple checksum formula used to validate various identification numbers. Created by Hans Peter Luhn at IBM in 1954, it's designed to protect against accidental errors, not malicious attacks.

### How It Works

#### Basic Steps:
1. **Start from the rightmost digit** (excluding check digit)
2. **Double every second digit** moving left
3. **If doubling results in >9, add the digits** (e.g., 14 → 1 + 4 = 5)
4. **Sum all digits**
5. **Check digit makes total sum multiple of 10**

#### Example: Validate `79927398713`

**Step 1: Process digits from right**
```
Number:   7   9   9   2   7   3   9   8   7   1   3
Double?:  ✓   ✗   ✓   ✗   ✓   ✗   ✓   ✗   ✓   ✗   ✓  (from right)
```

**Step 2: Double and process**
- 7×2=14 → 1+4=5
- 9×2=18 → 1+8=9  
- 7×2=14 → 1+4=5
- 9×2=18 → 1+8=9
- 7×2=14 → 1+4=5

**Step 3: Sum all digits**
```
Doubled:  5   9   9   2   5   3   9   8   5   1   3
Sum: 5+9+9+2+5+3+9+8+5+1+3 = 59
```

**Step 4: Check validity**
`59 % 10 = 9` ≠ 0 → **INVALID**

### Real-World Applications
- ✅ Credit/Debit card numbers
- ✅ IMEI numbers (mobile devices)
- ✅ National ID numbers (some countries)
- ✅ Gift card numbers

---

## 🏦 Modulo 97 Algorithm

### Overview
The **Modulo 97 algorithm** is a robust checksum method used primarily in banking systems. It provides much stronger error detection than Luhn and is the standard for European bank accounts and international transfers.

### How It Works

#### Basic Principle:
A number is valid if:  
**`(Large_Number) MOD 97 = 1`**

#### French RIB Structure:
```
BBBBB SSSSS AAAAAAAAAAA KK
└─┬─┘ └─┬─┘ └────┬────┘ └┬┘
Bank  Branch  Account   RIB Key  
Code   Code   Number   (2 digits)
```

### Important: Understanding RIB Components

**Where Letters Come From:**
- **Bank codes (5 digits)**: Always numbers
- **Branch codes (5 digits)**: Always numbers  
- **Account numbers (11 characters)**: Can contain letters A-Z
- **Letters conversion**: A=1, B=2, ..., Z=26
- **Zeros as padding**: `0000157841Z` means the actual account identifier is shorter, padded with leading zeros

### RIB Key Generation - Complete Example

**Scenario: Opening a new bank account**

**Given:**
- Bank Code: `12345`
- Branch Code: `67890`
- Account Number: `AB123CD` ← Customer identifier (contains letters)

**Step 1: Convert letters to numbers and pad to 11 digits**
- Convert: A=1, B=2, C=3, D=4
- Original: `AB123CD` → `1 2 1 2 3 3 4`
- Pad with zeros: `00001212334` ← Now 11 digits total

**Step 2: Build the giant number**
```
Bank:    12345
Branch:  67890  
Account: 00001212334
Concatenate: 123456789000001212334
```

**Step 3: Calculate modulo 97**
```
123456789000001212334 ÷ 97 
Remainder = 36
```

**Step 4: Calculate RIB Key**
```
RIB Key = 97 - 36 = 61
```

**Final RIB:** `12345 67890 00001212334 61`

### Validation Process
To validate a RIB, append the key and check modulo 97:

**Given RIB:** `12345 67890 00001212334 61`

1. Build number: `12345678900000121233461`
2. Check: `12345678900000121233461 % 97 = 1` ✅
3. If result = 1 → **VALID**, otherwise → **INVALID**

---

## 🌍 IBAN - International Bank Account Number

### Overview
IBAN extends Modulo 97 for international transfers. It contains:
- Country code + Check digits + Domestic account details

### French IBAN Example:
```
FR76 3000 4004 1000 0015 7841 Z02 02
└┬┘ └┬┘ └──────────────────────┬────────────────────┘
 │   │                         │
Country Check Domestic Account (RIB without spaces)
 Code Digits
```

### IBAN Validation Process

**Step 1: Rearrange**
Take: `FR76 3000 4004 1000 0015 7841 Z02`
Remove spaces: `FR7630004004100000157841Z02`
Move first 4 chars to end: `30004004100000157841Z02FR76`

**Step 2: Convert letters to numbers**
- A=10, B=11, ..., Z=35
- F=15, R=27
- `FR76` → `15 27 7 6`
- Final number: `30004004100000157841Z02152776`

**Step 3: Validate**
`30004004100000157841Z02152776 % 97 = 1` ✅

---

## ⚖️ Comparison

| Feature | Luhn Algorithm | Modulo 97 Algorithm |
|---------|----------------|---------------------|
| **Complexity** | Simple | Complex |
| **Error Detection** | Good (most single-digit errors) | Excellent (virtually all errors) |
| **Check Digits** | 1 digit | 2 digits |
| **Performance** | Fast | Slower (handles large numbers) |
| **Primary Use** | Consumer verification | Financial transactions |
| **Security** | Basic error checking | High-reliability checking |

---

## 🌍 Use Cases

### Luhn Algorithm
- **Credit Card Processing**: First-line validation before bank authorization
- **Mobile Devices**: IMEI number validation
- **ID Systems**: Various national identification numbers
- **Retail**: Gift cards, membership cards

### Modulo 97 Algorithm  
- **International Banking**: IBAN validation for cross-border transfers
- **French Banking**: RIB validation for domestic accounts
- **Financial Systems**: High-value transaction verification
- **Government**: Tax ID numbers in some countries

---

## 💻 Implementation Examples

### Luhn Algorithm (Python)
```python
def calculate_luhn_check_digit(number: str) -> int:
    def split_into_digits(n: Union[str, int]) -> List[int]:
        return [int(digit) for digit in str(n)]
    
    digits = split_into_digits(number)
    to_double_digit = digits[-1::-2] 
    non_double_digits =digits[-2::-2]
    total = sum(non_double_digits)

    for d in to_double_digit:
        doubled = d * 2
        total += sum(split_into_digits(doubled))

    return (10 - (total % 10)) % 10
```
### Or you can try this one
```python
def luhn_checksum(number_str):
    total = 0
    reverse_digits = number_str[::-1]
    for i, digit in enumerate(reverse_digits):
        n = int(digit)
        if i % 2 == 1:  # Double every second digit
            n *= 2
            if n > 9:
                n -= 9
        total += n
    return total % 10 == 0

# Example usage
credit_card = "4111111111111111"
print(f"Valid: {luhn_checksum(credit_card)}")  # True
```


### Modulo 97 Algorithm (Python)
```python
def rib_key(bank_code, branch_code, account_number):
    """Calculate RIB key for given components"""
    # Convert letters in account number (A=1, B=2, ..., Z=26)
    converted_account = ''
    for char in account_number:
        if char.isalpha():
            converted_account += str(ord(char.upper()) - 64)
        else:
            converted_account += char
    
    # Build the giant number
    giant_number = bank_code + branch_code + converted_account
    
    # Calculate RIB key
    remainder = int(giant_number) % 97
    return 97 - remainder

def validate_rib(bank_code, branch_code, account_number, key):
    """Validate a complete RIB"""
    # Convert letters in account number
    converted_account = ''
    for char in account_number:
        if char.isalpha():
            converted_account += str(ord(char.upper()) - 64)
        else:
            converted_account += char
    
    # Build complete number with key
    full_number = bank_code + branch_code + converted_account + str(key).zfill(2)
    
    # Check if valid
    return int(full_number) % 97 == 1

# Example usage
bank = "12345"
branch = "67890"
account = "AB123CD"
key = rib_key(bank, branch, account)
print(f"RIB Key: {key:02d}")  # 61

# Validate
is_valid = validate_rib(bank, branch, account, key)
print(f"RIB is valid: {is_valid}")  # True
```

---

## 📚 Notes

### Key Points to Remember:
- **Luhn**: "Quick typo-check for credit cards using digit doubling"
- **Modulo 97**: "Strong error detection for bank accounts using remainder = 1 rule"
- **Account letters**: Convert A=1, B=2, ..., Z=26 for RIB calculations
- **Zeros**: Used as padding to maintain fixed field lengths
- **IBAN**: International version that includes country codes

### Common Questions:
1. **"How would you validate a credit card number?"**
   - "I'd implement the Luhn algorithm: double every second digit from the right, sum all digits, and check if the total is divisible by 10."

2. **"What's the difference between Luhn and Modulo 97?"**
   - "Luhn is for quick consumer-facing checks like credit cards, while Modulo 97 is for high-reliability financial transactions like bank transfers."

3. **"Why do French RIBs have letters in account numbers?"**
   - "The letters are part of the customer identifier and get converted to numbers (A=1, B=2, etc.) for the checksum calculation, while maintaining human-readable account references."

4. **"Walk me through RIB key calculation"**
   - "Convert any letters to numbers, concatenate all components, calculate modulo 97, and subtract the remainder from 97 to get the 2-digit key."

---
