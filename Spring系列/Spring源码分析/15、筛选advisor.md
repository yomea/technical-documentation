<p>
&nbsp;&nbsp;&nbsp;&nbsp;spring获取到所有需要advisor后，并不是每个advisor都适用于当前bean，它需要经过筛选，过滤掉不适用的advisor，spring的切点匹配模式非常复杂，使用了解释器模式
</p>


```
protected List<Advisor> org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator.findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
        //设置当前正在代理的beanName
		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
		    //获取能够应用到当前bean的advisor
		    //(*1*)
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}
	
	//(*1*)
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
		//循环所有获取到的Advisor
		for (Advisor candidate : candidateAdvisors) {
		    //如果当前advisor是引介advisor那么判断当前这个引介advisor是否适用于当前bean
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
		    //如果是IntroductionAdvisor，那么已经处理过了
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			//匹配其他的advisor
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		//返回合适的advisor
		return eligibleAdvisors;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;在分析切点的解析时，我们先来看看在spring切点表达式中常用到的一些标识
</p>

标识 | 解释
---|---
arg() |	限制连接点匹配参数为指定类型的执行方法
@args() |	限制连接点匹配参数由指定注解标注的执行方法
execution() |	用于匹配是连接点的执行方法
this() |	限制连接点匹配AOP代理的bean引用为指定类型的类
target |	限制连接点匹配目标对象为指定类型的类
@target()|	限制连接点匹配特定的执行对象，这些对象对应的类要具有指定类型的注解
within()|	限制连接点匹配指定的类型
@within()|	限制连接点匹配指定注解所标注的类型（当使用SpringAOP时，方法定义在由指定的注解所标注的类里）
@annotation|	限定匹配带有指定注解的连接点


<p>
&nbsp;&nbsp;&nbsp;&nbsp;在匹配advisor，spring使用PointParse解析类型模式，假设此时我们设置的表达式为execution(public com.test.xixi.haha.*(..)) && args(a)
spring解析切点表达式使用的是PointParse
</p>


```
//构造函数
public PatternParser(String data) {
		this(BasicTokenSource.makeTokenSource(data, null));
}

//解析表达式，变成token片段
public static ITokenSource makeTokenSource(String input, ISourceContext context) {
		char[] chars = input.toCharArray();
		
		int i = 0;
		List<BasicToken> tokens = new ArrayList<BasicToken>();
		
		while (i < chars.length) {
			char ch = chars[i++];			
			switch(ch) {
				case ' ':
				case '\t':
				case '\n':
				case '\r':
					continue;//忽略掉空白字符
				case '*':
				case '(':
				case ')':
				case '+':
				case '[':
				case ']':
				case ',':
				case '!':
				case ':':
				case '@':
				case '<':
				case '>':
				case '=':
				case 	'?':
				    //解析操作符，使用BasicToken记录操作符和其在原始表达式中的起始位置和结束位置
				    tokens.add(BasicToken.makeOperator(makeString(ch), i-1, i-1));
				    continue;
				case '.':
					if ((i+2)<=chars.length) {
						// could be '...'
						char nextChar1 = chars[i];
						char nextChar2 = chars[i+1];
						if (ch==nextChar1 && ch==nextChar2) {
							// '...'
							//解析标识符，这是一种特殊的标识符，表示匹配任意字符
							tokens.add(BasicToken.makeIdentifier("...",i-1,i+1));
							i=i+2;
						} else {
						    //如果不是连续的三个点，那么就是一种普通的标识符
							tokens.add(BasicToken.makeOperator(makeString(ch), i-1, i-1));
						}
					} else {
				    	//如果不是连续的三个点，那么就是一种普通的标识符
						tokens.add(BasicToken.makeOperator(makeString(ch), i-1, i-1));
					}
					continue;
				case '&':
				    //解析与操作符
					if ((i+1) <= chars.length && chars[i] != '&') {
						tokens.add(BasicToken.makeOperator(makeString(ch),i-1,i-1));
						continue;
					}
					// fall-through
				case '|':
				    //解析或操作符
				    if (i == chars.length) {
				    	throw new BCException("bad " + ch);
				    }
				    char nextChar = chars[i++];
				    if (nextChar == ch) {
				    	tokens.add(BasicToken.makeOperator(makeString(ch, 2), i-2, i-1));
				    } else {
				    	throw new RuntimeException("bad " + ch);
				    }
				    continue;
				    
				case '\"':
				    //解析字符串，使用引号引用的字符串
				    int start0 = i-1;
				    while (i < chars.length && !(chars[i]=='\"')) i++;
				    i += 1;
				    tokens.add(BasicToken.makeLiteral(new String(chars, start0+1, i-start0-2), "string", start0, i-1));
				    continue;
				default:
				    //其他的字符都认为是标识符，比如上面例子中的execution，com等等
				    int start = i-1;
				    while (i < chars.length && Character.isJavaIdentifierPart(chars[i])) { i++; }
				    tokens.add(BasicToken.makeIdentifier(new String(chars, start, i-start), start, i-1));
				
			}
		}

		//System.out.println(tokens);
		
		return new BasicTokenSource((IToken[])tokens.toArray(new IToken[tokens.size()]), context);
	}


public org.aspectj.weaver.patterns.PatternParser.Pointcut parsePointcut() {
        //(*1*)
		Pointcut p = parseAtomicPointcut();
		//如果后面还有&&连接，那么使用AndPointcut连接
		if (maybeEat("&&")) {
			p = new AndPointcut(p, parseNotOrPointcut());
		}
        //如果是||,那么用于OrPointcut连接，递归解析
		if (maybeEat("||")) {
			p = new OrPointcut(p, parsePointcut());
		}

		return p;
	}
	
	//(*1*)
	private Pointcut parseAtomicPointcut() {
	    //按照我们假设的表达式，那么符合以！开头
		if (maybeEat("!")) {
		    //获取！在原始字符串的开始位置
			int startPos = tokenSource.peek(-1).getStart();
			//递归解析，使用取非切点包装
			Pointcut p = new NotPointcut(parseAtomicPointcut(), startPos);
			return p;
		}
		//解析括号中的表达式，使用顶层解析方法parsePointcut递归解析
		if (maybeEat("(")) {
			Pointcut p = parsePointcut();
			eat(")");
			return p;
		}
		//如果是@开始，那么这是一个注解切点
		if (maybeEat("@")) {
		    //注解类型在原始字符串的开始下标
			int startPos = tokenSource.peek().getStart();
			//解析注解
			Pointcut p = parseAnnotationPointcut();
			//)的结束位置
			int endPos = tokenSource.peek(-1).getEnd();
			//记录当前切点片段在原始字符串的开始位置与结束位置
			p.setLocation(sourceContext, startPos, endPos);
			return p;
		}
		//如果是普通表达式，就像我们上面定义的那个
		int startPos = tokenSource.peek().getStart();
		//用普通的方式进行解析
		Pointcut p = parseSinglePointcut();
		//获取当前切点的结束位置
		int endPos = tokenSource.peek(-1).getEnd();
		//记录当前切点片段在原始资源中的开始位置与结束位置
		p.setLocation(sourceContext, startPos, endPos);
		return p;
	}

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;很明显我们举例子用的表达式 execution(public com.test.xixi.haha.*(..)) && args(a) 是一个普通的表达式，那么直接关注以下代码
//这段表达式解析后的片段为execution ( public com . test . xixi . haha . * ( . . ) ) && args ( a )
</p>


```
private Pointcut parseAtomicPointcut() {
	    。。。。。。省略部分代码
		//如果是普通表达式，就像我们上面定义的那个,这里表示第一个execution在原始表达式的起始位置
		int startPos = tokenSource.peek().getStart();
		//用普通的方式进行解析
		//(*1*)
		Pointcut p = parseSinglePointcut();
		//获取当前切点的结束位置
		int endPos = tokenSource.peek(-1).getEnd();
		//记录当前切点片段在原始资源中的开始位置与结束位置
		p.setLocation(sourceContext, startPos, endPos);
		return p;
	}
	
		
	//(*1*)
	public Pointcut parseSinglePointcut() {
	    //execution的起始位置0
		int start = tokenSource.getIndex();
		//execution
		IToken t = tokenSource.peek();
		//没有实现，直接返回的是null，估计是用于缓存解析好的切点对象
		Pointcut p = t.maybeGetParsedPointcut();
		if (p != null) {
			tokenSource.next();
			return p;
		}
        //解析标识符
        //(*4*)
		String kind = parseIdentifier();
        //判断是哪种特殊的标识符，这个分支中的标识符类型我只认识execution，其他的call，get，set没用过
		if (kind.equals("execution") || kind.equals("call") || kind.equals("get") || kind.equals("set")) {
			p = parseKindedPointcut(kind);
			//解析参数名
		} else if (kind.equals("args")) {
			p = parseArgsPointcut();
			//this，指定代理类的类型
		} else if (kind.equals("this")) {
			p = parseThisOrTargetPointcut(kind);
			//指定被代理对象的类型
		} else if (kind.equals("target")) {
			p = parseThisOrTargetPointcut(kind);
			//指定被代理目标类的类型，多个逗号分割
		} else if (kind.equals("within")) {
			p = parseWithinPointcut();
			//指定被代理目标类的方法，多个逗号分割
		} else if (kind.equals("withincode")) {
			p = parseWithinCodePointcut();
		} else if (kind.equals("cflow")) {
			p = parseCflowPointcut(false);
		} else if (kind.equals("cflowbelow")) {
			p = parseCflowPointcut(true);
		} else if (kind.equals("adviceexecution")) {
			eat("(");
			eat(")");
			p = new KindedPointcut(Shadow.AdviceExecution, new SignaturePattern(Member.ADVICE, ModifiersPattern.ANY,
					TypePattern.ANY, TypePattern.ANY, NamePattern.ANY, TypePatternList.ANY, ThrowsPattern.ANY,
					AnnotationTypePattern.ANY));
		} else if (kind.equals("handler")) {
			eat("(");
			TypePattern typePat = parseTypePattern(false, false);
			eat(")");
			p = new HandlerPointcut(typePat);
		} else if (kind.equals("lock") || kind.equals("unlock")) {
			p = parseMonitorPointcut(kind);
		} else if (kind.equals("initialization")) {
			eat("(");
			SignaturePattern sig = parseConstructorSignaturePattern();
			eat(")");
			p = new KindedPointcut(Shadow.Initialization, sig);
		} else if (kind.equals("staticinitialization")) {
			eat("(");
			TypePattern typePat = parseTypePattern(false, false);
			eat(")");
			p = new KindedPointcut(Shadow.StaticInitialization, new SignaturePattern(Member.STATIC_INITIALIZATION,
					ModifiersPattern.ANY, TypePattern.ANY, typePat, NamePattern.ANY, TypePatternList.EMPTY, ThrowsPattern.ANY,
					AnnotationTypePattern.ANY));
		} else if (kind.equals("preinitialization")) {
			eat("(");
			SignaturePattern sig = parseConstructorSignaturePattern();
			eat(")");
			p = new KindedPointcut(Shadow.PreInitialization, sig);
		} else if (kind.equals("if")) {
			eat("(");
			if (maybeEatIdentifier("true")) {
				eat(")");
				p = new IfPointcut.IfTruePointcut();
			} else if (maybeEatIdentifier("false")) {
				eat(")");
				p = new IfPointcut.IfFalsePointcut();
			} else {
				if (!maybeEat(")")) {
					throw new ParserException(
							"in annotation style, if(...) pointcuts cannot contain code. Use if() and put the code in the annotated method",
							t);
				}
				// TODO - Alex has some token stuff going on here to get a readable name in place of ""...
				p = new IfPointcut("");
			}
		} else {
		    //解析自定义切点类型，需要实现PointcutDesignatorHandler接口
			boolean matchedByExtensionDesignator = false;
			// see if a registered handler wants to parse it, otherwise
			// treat as a reference pointcut
			for (PointcutDesignatorHandler pcd : pointcutDesignatorHandlers) {
				if (pcd.getDesignatorName().equals(kind)) {
					p = parseDesignatorPointcut(pcd);
					matchedByExtensionDesignator = true;
				}

			}
			//如果没有自定义的匹配类型，那么可能是引用的切点，比如我们在注解中常用的引用某个方法上的切点
			if (!matchedByExtensionDesignator) {
				tokenSource.setIndex(start);
				p = parseReferencePointcut();
			}
		}
		return p;
	}
	
	//(*4*)
	public String parseIdentifier() {
		IToken token = tokenSource.next();
		//如果是标识符，那么直接返回
		if (token.isIdentifier()) {
			return token.getString();
		}
		throw new ParserException("identifier", token);
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面有好多我们平时根本就没有使用过的切点类型，本人表示惭愧，我迄今为止，在项目中只使用过execution，甚至有些类型，我根本不知道啥意思,本人暂时也不想去管诸如cflowbelow，cflow这些类型的切点。我们继续分析下execution是如何解析的
</p>


```
private KindedPointcut parseKindedPointcut(String kind) {
        //吃掉（字符，如果不是（字符，不好意思，老子用异常甩你一脸
		eat("(");
		SignaturePattern sig;

		Shadow.Kind shadowKind = null;
		if (kind.equals("execution")) {
		    //解析方法或者构造器签名
		    //(*1*)
			sig = parseMethodOrConstructorSignaturePattern();
			//如果是方法，标识为方法
			if (sig.getKind() == Member.METHOD) {
				shadowKind = Shadow.MethodExecution;
				//如果是构造器，那么表示为构造器
			} else if (sig.getKind() == Member.CONSTRUCTOR) {
				shadowKind = Shadow.ConstructorExecution;
			}
		} 
		。。。。。。省略部分代码
		
		eat(")");
		return new KindedPointcut(shadowKind, sig);
	}
	
	//(*1*)
	public SignaturePattern parseMethodOrConstructorSignaturePattern() {
	    //获取（后面第一个token的开始位置
		int startPos = tokenSource.peek().getStart();
		//解析注解，可能会有execution(@org.springframework.beans.factory.annotation.Autowired public com.test.xixi.haha.*(..))
		//这种类型的表达式，这里面解析注解的方式和上面解析注解的方式不同，这里面的逻辑相对比较复杂
		//看代码时，发现它会解析诸如@(org.springframework.beans.factory.annotation.Autowired)这种形式的，不懂这是啥表达式
		//知道的同学没法教教我
		//此处解析Autowired注解
		AnnotationTypePattern annotationPattern = maybeParseAnnotationPattern();
		//解析修饰符，比如此处的public，封装成ModifiersPattern，匹配的时匹配其修饰符是否为public即可
		//此外如果我们定义的!public，那么会匹配要求的修饰符然后再匹配禁止的修饰符，比如表达式写成 public !public 那么这样的表达式
		//永远无法匹配
		ModifiersPattern modifiers = parseModifiersPattern();
		//解析返回类型，这里面解析的逻辑也非常复杂，这个方法可单独用于引介增强，用于类型的匹配
		TypePattern returnType = parseTypePattern(false, false);

		TypePattern declaringType;
		NamePattern name = null;
		MemberKind kind;
		// here we can check for 'new'
		//匹配最后一个方法名是否为new，表示构造方法
		if (maybeEatNew(returnType)) {
			kind = Member.CONSTRUCTOR;
			if (returnType.toString().length() == 0) {
				declaringType = TypePattern.ANY;
			} else {
				declaringType = returnType;
			}
			returnType = TypePattern.ANY;
			name = NamePattern.ANY;
		} else {
		    //方法
			kind = Member.METHOD;
			//下一token，这里是com
			IToken nameToken = tokenSource.peek();
			//解析方法所在类，按照我们的例子，那么这个类型是com.zhipin.service.*.*
			declaringType = parseTypePattern(false, false);
			if (maybeEat(".")) {
			    //如果后面是点，那么获取点后面的值为方法名
				nameToken = tokenSource.peek();
				name = parseNamePattern();
			} else {
			    //获取最后一个点后面的名字作为方法名
				name = tryToExtractName(declaringType);
				if (declaringType.toString().equals("")) {
					declaringType = TypePattern.ANY;
				}
			}
			if (name == null) {
				throw new ParserException("name pattern", tokenSource.peek());
			}
			String simpleName = name.maybeGetSimpleName();
			// XXX should add check for any Java keywords
			if (simpleName != null && simpleName.equals("new")) {
				throw new ParserException("method name (not constructor)", nameToken);
			}
		}
        //解析参数
		TypePatternList parameterTypes = parseArgumentsPattern(true);
        //解析异常列表
		ThrowsPattern throwsPattern = parseOptionalThrowsPattern();
		//将解析好的数据用SignaturePattern封装
		SignaturePattern ret = new SignaturePattern(kind, modifiers, returnType, declaringType, name, parameterTypes,
				throwsPattern, annotationPattern);
		int endPos = tokenSource.peek(-1).getEnd();
		ret.setLocation(sourceContext, startPos, endPos);
		return ret;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;假设我们设置的切点表达式是@annotation(com.zhiping.annotation.Test)
</p>


```
public Pointcut parseAnnotationPointcut() {
		int start = tokenSource.getIndex();
		IToken t = tokenSource.peek();
		String kind = parseIdentifier();
		IToken possibleTypeVariableToken = tokenSource.peek();
		String[] typeVariables = maybeParseSimpleTypeVariableList();
		if (typeVariables != null) {
			String message = "(";
			assertNoTypeVariables(typeVariables, message, possibleTypeVariableToken);
		}
		tokenSource.setIndex(start);
		
		//@annotation 代理被指定注解修饰的连接点
		if (kind.equals("annotation")) {
			return parseAtAnnotationPointcut();
			//@args 代理被指定注解修饰的连接点方法参数
		} else if (kind.equals("args")) { 
			return parseArgsAnnotationPointcut();
			//@this 使用被指定注解修饰的代理类代理,@target 代理被指定注解修饰的目标类
		} else if (kind.equals("this") || kind.equals("target")) {
			return parseThisOrTargetAnnotationPointcut();
			//@within 代理被指定注解中的任意一个注解修饰的目标类
		} else if (kind.equals("within")) {
			return parseWithinAnnotationPointcut();
			//@withincode
		} else if (kind.equals("withincode")) {
			return parseWithinCodeAnnotationPointcut();
		}
		throw new ParserException("pointcut name", t);
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;假设我们设置的切点表达式是@annotation(com.zhiping.annotation.Test)
</p>


```
private Pointcut parseAtAnnotationPointcut() {
        //解析标识符，这里解析后的标识符是annotation
		parseIdentifier();
		eat("(");
		if (maybeEat(")")) {
			throw new ParserException("@AnnotationName or parameter", tokenSource.peek());
		}
		//解析注解，封装成注解类型模式
		//(*1*)
		ExactAnnotationTypePattern type = parseAnnotationNameOrVarTypePattern();
		eat(")");
		return new AnnotationPointcut(type);
	}
	
	//(*1*)
	protected ExactAnnotationTypePattern parseAnnotationNameOrVarTypePattern() {
		ExactAnnotationTypePattern p = null;
		int startPos = tokenSource.peek().getStart();
		if (maybeEat("@")) {
			throw new ParserException("@Foo form was deprecated in AspectJ 5 M2: annotation name or var ", tokenSource.peek(-1));
		}
		//解析注解名，封装成ExactAnnotationTypePattern
		//(*2*)
		p = parseSimpleAnnotationName();
		int endPos = tokenSource.peek(-1).getEnd();
		p.setLocation(sourceContext, startPos, endPos);
		// For optimized syntax that allows binding directly to annotation values (pr234943)
		if (maybeEat("(")) {
			String formalName = parseIdentifier();
			p = new ExactAnnotationFieldTypePattern(p, formalName);
			eat(")");
		}
		return p;
	}
	
	//(*2*)
	private ExactAnnotationTypePattern parseSimpleAnnotationName() {
		// the @ has already been eaten...
		ExactAnnotationTypePattern p;
		StringBuffer annotationName = new StringBuffer();
		//解析注解标识符，最终变成com.zhiping.annotation.Test
		annotationName.append(parseIdentifier());
		while (maybeEat(".")) {
			annotationName.append('.');
			annotationName.append(parseIdentifier());
		}
		UnresolvedType type = UnresolvedType.forName(annotationName.toString());
		p = new ExactAnnotationTypePattern(type, null);
		return p;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;切点表达式的解析就到这里，如果对更多的细节感兴趣的话，请自行学习。
</p>









