# 2.1
```
if (isalpha(lastChar)) {
    std::string ident = "";
    ident += lastChar;
    while(isalpha(lastChar = getNextChar(iFile)))
        ident += lastChar;
    if (ident == "def")
        return tok_def;
    setIdentifier(ident);
    return tok_identifier;
}
```
- 指示通りに実装する。
