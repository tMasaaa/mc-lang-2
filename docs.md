# 2.1
- 指示通りに実装する。
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

# 2.2
- 指示通りに実装する。
- しばらくParseIdentifierExpr自体が呼ばれず困っていたが、それはParsePrototypeでの間違いが原因だった。デバッグ方法がプリントデバッグしか知らないので、他の方法がないかと思った。(例えば、特定の関数がCallされているかどうか調べるなど)
```
    // return nullptr;
    // 1. getIdentifierを用いて識別子を取得する。
    std::string IdName = lexer.getIdentifier();
    // std::cout<<"Id?"<<IdName<<std::endl; // ?
    // 2. トークンを次に進める。
    getNextToken();
    // 3. 次のトークンが'('の場合は関数呼び出し。そうでない場合は、
    // VariableExprASTを識別子を入れてインスタンス化し返す。
    if (CurTok != '(')
        return llvm::make_unique<VariableExprAST>(IdName);
    // 4. '('を読んでトークンを次に進める。
    getNextToken();
    // 5. 関数fooの呼び出しfoo(3,4,5)の()内をパースする。
    // 引数は数値、二項演算子、(親関数で定義された)引数である可能性があるので、
    // ParseExpressionを用いる。
    // 呼び出しが終わるまで(CurTok == ')'になるまで)引数をパースしていき、都度argsにpush_backする。
    // 呼び出しの終わりと引数同士の区切りはCurTokが')'であるか','であるかで判別できることに注意。
    std::vector<std::unique_ptr<ExprAST>> args;
    if (CurTok != ')') {
        while(true) {
            if (auto arg = ParseExpression())
                args.push_back(std::move(arg));
            else
                return nullptr;
            if (CurTok == ')')
                break;
            if (CurTok != ',')
                return LogError("Expected `)` or `,` in arg list in ParseIdentifierExpr().");
            getNextToken(); // skip `,`, go next arg
        }
    }
    // 6. トークンを次に進める。
    getNextToken(); // skip `)`
    // 7. CallExprASTを構成し、返す。
    return llvm::make_unique<CallExprAST>(IdName, std::move(args));
```

# 2.3
- これも指示通りに実装する。
- whileの中でgetNextToken()を二回呼んでしまって、最初は`(`をスキップしてうまく行くけど次に`)`が来ると困るみたいな挙動で、lexer, parserどちらがおかしいのかしばらく分からず30分ほど苦労した。ここでは、最初の`(`だけをスキップしたいので、whileの外で呼んでやる必要がある。
```
static std::unique_ptr<PrototypeAST> ParsePrototype() {
    // 2.2とほぼ同じ。CallExprASTではなくPrototypeASTを返し、
    // 引数同士の区切りが','ではなくgetNextToken()を呼ぶと直ぐに
    // CurTokに次の引数(もしくは')')が入るという違いのみ。
    // return nullptr;
    if (CurTok != tok_identifier)
        return LogErrorP("Expected function name in ParsePrototype().");
    std::string fn_name = lexer.getIdentifier();
    getNextToken();
    if (CurTok != '(')
        return LogErrorP("Expected `(` in ParsePrototype().");
    std::vector<std::string> arg_names;
    getNextToken(); // skip `(`
    while (true) {
        // std::cout<<"proto?"<<CurTok<<std::endl;
        if (CurTok == ')')
            break;
        if (CurTok != tok_identifier)
            return LogErrorP("Expected Identifier in ParsePrototype().");
        arg_names.push_back(lexer.getIdentifier());
        getNextToken();
    }
    getNextToken();
    return llvm::make_unique<PrototypeAST>(fn_name, std::move(arg_names));
}
```

# 2.4
- 引数名がvariableASTで、コメントではNameだったので少し戸惑いました。
- 指示通りに実装する。ここでまだ実は2.3のバグが修正できてなくて、あれ、codegenにもバグ埋め込んだかなと思って困っていました。
```
Value *VariableExprAST::codegen() {
    // return nullptr;
    // NamedValuesの中にVariableExprAST::NameとマッチするValueがあるかチェックし、
    // あったらそのValueを返す。
    // std::cout<<"vAST"<<NamedValues<<std::endl;
    Value *V = NamedValues[variableName];
    if (!V)
        return LogErrorV("Unknown variable name in VariableExprAST::codegen().");
    return V;
}
```

# 2.5
- これも指示通りに実装する。
- `return nullptr`をコメントアウトし忘れて少し困りました。
- エラーメッセージは、最終的には簡潔にすべきですが、開発段階ではどの関数で失敗したか示すために`in fumc`みたいなのを入れると捗るなと思いました。
```
// TODO 2.5: 関数呼び出しのcodegenを実装してみよう
Value *CallExprAST::codegen() {
    // return nullptr;
    // 1. myModule->getFunctionを用いてcalleeがdefineされているかを
    // チェックし、されていればそのポインタを得る。
    Function *callee_function = myModule->getFunction(callee);
    if (!callee_function)
        return LogErrorV("Unknown function referenced in CallExprAST::codegen().");
    // 2. llvm::Function::arg_sizeと実際に渡されたargsのサイズを比べ、
    // サイズが間違っていたらエラーを出力。
    if (callee_function->arg_size() != args.size())
        return LogErrorV("Incorrect number of arguments in CallExprAST::codegen().");
    std::vector<Value *> argsV;
    // 3. argsをそれぞれcodegenしllvm::Valueにし、argsVにpush_backする。
    for (unsigned i = 0, e = args.size(); i != e; ++i) {
        argsV.push_back(args[i]->codegen());
        if (!argsV.back())
            return nullptr;
    }
    // 4. IRBuilderのCreateCallを呼び出し、Valueをreturnする。
    return Builder.CreateCall(callee_function, argsV, "calltmp");
}
```
# 2.6
- これかなり感動しました。ELF見ても何も分からなかった。
- `objdump -d main`の一部
```
00000000004008b0 <foobar>:
  4008b0:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4008b4:	48 29 d0             	sub    %rdx,%rax
  4008b7:	c3                   	retq
  4008b8:	90                   	nop
  4008b9:	90                   	nop
  4008ba:	90                   	nop
  4008bb:	90                   	nop
  4008bc:	90                   	nop
  4008bd:	90                   	nop
  4008be:	90                   	nop
  4008bf:	90                   	nop

00000000004008c0 <myfunc>:
  4008c0:	50                   	push   %rax
  4008c1:	ba 05 00 00 00       	mov    $0x5,%edx
  4008c6:	e8 e5 ff ff ff       	callq  4008b0 <foobar>
  4008cb:	59                   	pop    %rcx
  4008cc:	c3                   	retq
  4008cd:	0f 1f 00             	nopl   (%rax)
```
- これmyfuncが残っているんですね、感動しました。すごい。
