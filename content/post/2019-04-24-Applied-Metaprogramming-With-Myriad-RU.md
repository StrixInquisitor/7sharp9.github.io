+++
author = "7sharp9"
date = "2019-04-24"
translator = "StrixInquisitor"
description = "With Myriad And Falanx"
tags = ["FSharp", "Metaprogramming", "Quotations", "AST","Type Providers", "TAST"]
title = "Applied Meta-Programming"
+++
## Что такое Myriad?

[Myriad][17] это средство предварительной компиляции, который генерирует код на основе другого кода. Он интегрирован в цикл создания проекта через расширение MSBuild. Так же возможно использовать это его отдельно, как [средство интерфейса командной строки][21], но это сочинение(произведение?, статья?) разбереться с описанием интеграции в проект MSBuild, т.к. это наиболее встречающийся способ использования.

## Historical basis
До того как я углублюсь, будет к лучшему узнать некоторые исторические детали, такие как то, что проект [Myriad][17] проистекает из предыдущей работы по мета-программированию, над которой я трудился последний год, и возможно из еще более ранней истории использования мной F#. [Falanx][6] делает первые шаги к [Myriad][17] используя части функционала разных областей экосистемы F# и совместить их так, чтобы помочь в кодогенерации.Давайте пройдемся по нескольким различным способам использовать мета-программировании в F#.  
    

## Метапрограммирование 101
F# имеет неокторое количество метапрограммных средств, о которых я уже говорил ранее в моих предыдущих постах в блоге:  [Цитирование кода][15], [Провайдеры типов][9], [Типизированное выражение](https://fsharp.github.io/FSharp.Compiler.Service/typedtree.html), а так же [нетипизированное абстрактное синтактическое дерево - AST](https://fsharp.github.io/FSharp.Compiler.Service/untypedtree.html).  

### Цитирования 
[Цитирования][15] подкреплено рефлексией и в основном используются чтобы преобразовывать F# в другие языки. Они ограничены в этом, они не могут представить весь язык F#.  Типы и модули не могут быть представлены так же как и в AST F#. Они так же не кодируют F# на взаимно-однозначной основе с позиции представления F# выражений. Некоторые элементы, такие как декомпозиция размеченных объединений, сопоставление с образцом и неизменность(устойчивость?) функций не представлены в том же виде, в каком они представлены в F#, детали теряются при трансформации.  

### Провайдеры типов
[Провайдеры типов][9] use [цитирование][15] для кодирования информации метода и использовать эти выражения с скелетом типов создаваемый с помощью какой нибудь схемы или входных данных. [Провайдеры типов][9] могут быть полезы в ограниченном количистве сценариев. Создание [провайдеров типов][9]не должно быть воспринято легко, так как здесь может быть много граничных кейсов и отладок до того как они будут готовы. Так же нужно учесть, что [провайдеры типов][9] не могут создавать никакие F# конструкции, такие как [записи][23] или [размеченные объединения][22].  

### Типизированные выражения
Типизированные выражения используются для трансформации всего языка или всей системы и являются технологией, используемой в [Fable][7], чтобы транспилировать F# в JavaScript.  Вы можете прочитать больше о типизированных выражениях в моем [блоге][7]. Они не имеют зависимости на рефлексию и не требуют ни каких сборок на диске.  

### Нетипизированное AST
Нетипизированное AST или AST для краткости не та область, которая хорошо используется, в основном работа с AST довольно сложна вне компилятора и обычно AST создается компилятором, как результат фазы парсинга.  
- - -
## Уникальное путешествие в Falanx

Это не в рамках данного сочинения - описывать [Falanx][6] в деталях, но прочтение небольшой предыстории о нем может предоставить понимание о том курсе, который сформировался в [Myriad][17] и как некоторые произошедшие технические трудности, полученные при разработке [Falanx][6] повлияли на результирующие проектные решения. Ниже мы пробежимся через некоторые концепты и вопросы встречающиеся в [Falanx][6].  

[Falanx][6] использует смесь [Провайдеров типа][9], _(в основном [Type Provider SDK][8], а не механизм вызова провайдеров типов)_ [Цитирования][15], и манипуляции с AST для генерации исходного кода.  [Falanx][6] развился из минимально работающего продукта, в котором ключевыми входящеми факторами были прототипы, которые могли быть разработаны в течении нескольких недель, они могли брать схему [protocol buffer][10] и генерировать F# [Записи][23] и [Размеченных объединений][22] как исходящий файл.  

Это пример файла `[Protocol Buffer][10]:
 
```
syntax = "proto3";
message BundleRequest {
  int32 martId = 1;
  string member_id = 2;
}
```

Falanx работает с определения расширения MSBuild, которое ссылается на файл [Protocol Buffer][10] и генерирует в ответ содержащей F# [записи][23] and [размеченные объединения][22].  

```
<ItemGroup>
    <PackageReference Include="Falanx.Sdk" Version="0.4.*" PrivateAssets="All" />
</ItemGroup>

<ProtoFile Include="..\proto\bundle.proto">
    <OutputPath>mycustom.fs</OutputPath>
</ProtoFile>
```
`<ProtoFile Include="..\proto\bundle.proto">` is the input file and `<OutputPath>mycustom.fs</OutputPath>` is the output file.  

Результирующий сгенерированный исходный код выглядит так:

```fsharp
[<CLIMutable>]
type BundleRequest =
    { mutable martId : int option
      mutable memberId : string option }

    static member JsonObjCodec =
        fun martId memberId ->
        { martId = martId
          memberId = memberId }
        <!> Operators.jopt<BundleRequest, Int32> ("martId") (fun x -> x.martId)
        <*> Operators.jopt<BundleRequest, String> ("memberId") (fun x -> x.memberId)

    static member Serialize(m : BundleRequest, buffer : ZeroCopyBuffer) =
        writeOption<Int32>  (writeInt32)  (1) (buffer) (m.martId)
        writeOption<String> (writeString) (2) (buffer) (m.memberId)

    static member Deserialize(buffer : ZeroCopyBuffer) =
        deserialize<BundleRequest> (buffer)

    interface IMessage with
        member x.Serialize(buffer : ZeroCopyBuffer) =
            BundleRequest.Serialize(x, buffer)

        member x.ReadFrom(buffer : ZeroCopyBuffer) =
            let enumerator =
                ZeroCopyBuffer.allFields(buffer).GetEnumerator()
            while enumerator.MoveNext() do
                let current = enumerator.Current
                if current.FieldNum = 2 then x.memberId <- (Some(readString current) )
                else if current.FieldNum = 1 then x.martId <- (Some(readInt32 current) )
                else ()
            enumerator.Dispose()

        member x.SerializedLength() = serializedLength<BundleRequest> (x)
```
Первая часть сгенерированого кода представляет из себя определения [record][23] , за которыми следуют методы бинарной и json сериализаций. `JsonObjCodec` используется с помощью библиотеки [Fleece][11], `Serialize` и `Deserialize` используются библиотекой Froto[12]. [Записи][22] помогают и бинарной, и json сериализациям с помощью библиотек [Froto][12] и [Fleece][11] соответственно.  [Fleece][11] интересен тем, что сгенерированный код использует прикладной стиль и требует только кодек свойство 'JsonObjCodec' для сериализации и десериализации в json:

```fsharp
//serialize
let bundleRequest = {name = Some 42; memberId = Some "172c" }
printfn "%s" (string (toJson bundleRequest))

//deserialize
let bundleRequest2 = parseJson<BundleRequest> """{"martId": "54", "memberId": "172d"}"""
```

#### Парсинг/AST

Одной из технических трудностей [Falanx][6] было то, что нужно было создать минимально жизнеспособный проект за короткий промежуток времени. [Провайдеры типов][9] были вне обсуждения так как одним из основных требований было создание F# [records][23] и [Discriminated Unions][22].

Библиотека [Froto][12] уже имела работающие [Type Provider][9], но результирующий код были не F# [records][23] и [Discriminated Unions[22], так как [Type Providers][9] поддерживают только базовые CLR типы и не поддерживали никакие F# специфичные типы. Другой проблемой было то, что [Protocol Buffers][10] второй версии был единственной поддерживаемой версией, так что мы решили использовать парсер их [Froto][12]  и улучшить PR так, что бы он поддерживал синтаксис третьей версии.  

#### Цитирования
Мы сейчас имеем Protocol Buffer 3 файл представляющий абстрактное синтаксическое дерево. В [Froto][12] также существуют [цитирования][15] , которые были использованы для методов Проведенных типов, которые были использованны в [Type Provider][9] [Froto][12], хотя мы не использовали [Type Provider[9] в [Froto][12], мы можем использовать некоторые из [quotations][15] и адаптировать их для наших нужд. Дополнительные [quotations][15] были созданы, чтобы сформировать свойство `JsonObjCodec` из кода выше.  

#### Цитирования -> AST
Следующая часть заключалась в том, что [quotations][15] представляют `Serialize`, `Deserialize` и `JsonObjCodec` и преобразует их в исходный код. И цитирования и F# AST представляют похожие, но не одинаковые вещи: коллекция заметок которые представляют абстрактное понятие кода. [Quotations][15] не отображаются полностью в F# AST так как, они только представляют подмножество AST, типы для примера не представлены в [quotations][15], так же не могут цитирования отображать функции сопоставимые с F# AST заметкам, так как [quotations][15] не доставало деталей в сравнении с F# AST, таких как информации о фиксированности, паттерн матчинг и декомпозиция Размеченных объединений измененны в цитировании букв и композиции выражений цитирования.

Существует одна очень интересная библиотека под названием [Quotation Compiler][13] от[Eirik Tsarpalis](https://github.com/eiriktsarpalis), в этой библиотеке есть кусок кода, который использует точку входа в F# компиляторе, которая позволяет вам компилировать АСТ в длл вместо использования исходного файла. Я помню как смотрел на это раньше и задавался вопросом, можно ли использовать что-то подобное. Я так же нашел, что здесь есть функция, которая трансформирует [quotations][15] во фрагменты АСТ. Я понял то, что ее можно адаптировать так как мне нужно с небольшим количеством изменений. К сожалению было невозможно ссылаться на эту библиотеку на прямую так что, в конце концов мне пришлось сильно модифицировать ее для работы с [quotations][15] , которые ссылались на проведенные типы из за проблем с ссылками, но она формировала базис для основной части решения. Проблемы с ссылками связаны с тем как представлены [Type Providers][9]. Каждый тип созданный в генеративном провайдере типов поддержен субтипом одного из базовых типов рефлексии: 

Type|Base
----|----
ProvidedTypeSymbol | TypeDelegator
ProvidedSymbolMethod | MethodInfo
ProvidedStaticParameter | ParameterInfo
ProvidedParameter | ParameterInfo
ProvidedConstructor | ConstructorInfo
ProvidedMethod | MethodInfo
ProvidedProperty | PropertyInfo
ProvidedEvent | EventInfo
ProvidedField | FieldInfo
ProvidedTypeDefinition | TypeDelegator
MethodSymbol2 | MethodInfo
ConstructorSymbol | ConstructorInfo
MethodSymbol | MethodInfo
PropertySymbol | PropertyInfo
EventSymbol | EventInfo
FieldSymbol | FieldInfo
Type Symbol | TypeDelegator
TargetGenericParam | TypeDelegator
TargetTypeDefinition | TypeDelegator
TargetModule | Module
TargetAssembly | Assembly
ProvidedAssembly | Assembly

Это может начать становиться проблемой когда вы начинаете использовать ссылки на проведенные типы, так как [Type Provider SDK][8] выполняют минимальные перегрузки базовых типов так, что вы можете быстро встретиться с не выполнимым исключением довольно легко. Пример этого как если бы вы попытались получить доступ к свойству `ReflectedType` `ProvidedProperty`.
```fsharp
    let notRequired this opname item =
        let msg = sprintf "The operation '%s' on item '%s' should not be called on provided type, member or parameter of type '%O'. Stack trace:\n%s" opname item (this.GetType()) Environment.StackTrace
        Debug.Assert (false, msg)
        raise (NotSupportedException msg)

    type ProvidedProperty(...) =
        inherit PropertyInfo()
        ...
        override this.ReflectedType = notRequired this "ReflectedType" propertyName  
```
Все это окончилось тем, что требовались некоторые креативные обходы и обратнуая разработка функционала рефлексий в FSharp.Core, что заняло довольно много времени.
Пример этого при преобразовании выражения цитирования `NewRecord` в фрагмент AST:

```fsharp
    match expr with
    | NewRecord(ty, entries) ->
        let synTy = sysTypeToSynType range ty knownNamespaces ommitEnclosingType
        let fields =
            match ty with
            | :? ProvidedRecord as pr -> pr.RecordFields
            | _ -> FSharpType.GetRecordFields(ty, BindingFlags.NonPublic ||| BindingFlags.Public) |> Array.toList
        let synEntries = List.map exprToAst entries
        let entries = (fields, synEntries) ||> List.map2 (fun f e -> (mkLongIdent range [mkIdent range f.Name], true), Some e, None)
        let synExpr = SynExpr.Record(None, None, entries, range)
        SynExpr.Typed(synExpr, synTy, range)
```
Если вы посмотрите на `match ty with` вы можете увидеть, что здесь есть специальный match, когда `ty` это `ProvidedRecord`, который вызывает `pr.RecordFields` вместо `FSharpType.GetRecordFields`.   Это нужно потому, что `GetRecordFields` получает доступ к информации о рефлексии, что не встроенно полностью в [Type Provider SDK][8], из-за настраиваемых атрибутов, которые FSharp.Core должен предоставлять.

Другой аспект этого было трансформирование Проведенные типы в AST заметки. Так же как и встроенные типы в [Type Provider SDK][8], я добавил несколько новых типов: `ProvidedUnion`, `ProvidedUnionCase` и `ProvidedRecord`. Эти кастомные типы вместе с нормальными проведенными типами были перенесены в фрагменты АСТ вместе с цитированием, которые были трансформированны в заметки АСТ. Комбинация всех этих элементов была использована, чтобы сделать полное АСТ, которое состояло из модулей, записей с member functions и Размеченные объединения с member functions.

#### AST -> Исходный код
Нам не нужно, чтобы вывод был dll, но подобный актуальный исходный код был одним из основных требований, чтобы сделать это, я использовал библиотеку [Fantomas][4], которая позволяет вам использовать АСТ как входные данные и получать обратно исходный код предоставленный АСТ.

#### Обобщение всех трудностей с Falanx

Было немало трудностей с Falanx, в основном в использовании и [quotations][15].  В других языках цитирование и разцитирование одна из важнейших частей языка, однако, это не так с F#, так что существует много подводных камней при работе с [quotations][15].

*  [Quotations][15] действительно сложны при работе с ними, когда вам нужно создавать сложные функции, и будет еще сложнее, если вы смешаете [Statically Resolved Type Parameters][14] и рефлексию.  

*  [Quotations][15] теряют детали, когда вы используете литералы цитирования _(секции кода заключенные операторами `<@`/`@>` и `<@@`/`@@>`_). Например сопоставление шаблонов трансформируют в блоки if else, fixity information потеряна, так что вы не знаете, вызван ли оператор с инфиксной или префиксной нотацией. _(Я поднял некоторые из этих вопросов в качестве предложений языка F#: [Добавление сопоставления шаблона объединения в Expr](https://github.com/fsharp/fslang-suggestions/issues/681), [Добавление поддержки для записи fixity в цитированные литералы](https://github.com/fsharp/fslang-suggestions/issues/680))_

*  [Quotations][15] не представляет полный диапазон выражений возможных в F#. Типы и Модули не представлены, декомпозиция Размеченных объединений не имеет форм цитирования.

*  Трансформированные [quotations][15] не всегда являются элегантным идиоматическим кодом, в основном из-за причины, указанный выше, и того факта, что сопостовление шаблонов стирается и т.д. 

*  Splicing types при [quotations][15] могут быть действительно коварными, особенно если вы используете библиотеку, которая использует  много общих параметров как Fleece. Вывод типов в библиотеках как Fleece делает действительно приятным использование с кастомными [applicative operators](https://github.com/mausch/Fleece#codec), но определяя общие функции с 5 или более общими типами в сигнатуре типа совсем не весело.  Есть [approved language proposal](https://github.com/fsharp/fslang-suggestions/issues/670) добавлять splicing of types с оператором ~ , который мог бы помочь с этим как-нибудь.  

*  Работа с  Provided Types из [Type Provider SDK][8] может быть действительно вызывающим с [quotations][15], так как [quotations][15] цитирование поддержано рефлексией и Проведенные типы имеют встроенное минимальная рефлексия. Существует много случаев, где композирование Цитаты провалится во время выполнения из-за недостающей реализации рефлексии. Это иногда требует подключить либо реализацию рефлексий в [Type Provider SDK][8], либо кастомные рефлексии , чтобы извлекать информацию из Provided Types.  

В целом это был довольно напряженный процесс разработки с большим количеством отладок и копания в коде FSharp.Core и приходить к креативным решениям с проблемами рефлексии. Я возможно мог написать целую статью о подводных камнях, трюках и острых моментов при работе с цитированием, не выстрелив себе в ногу из мушкетона.
  
- - -
# Технический обзор Myriad
#### _Хорошо, хватит Falanx, что там с Myriad?_

[Myriad][17] схож с расширением [ppx_deriving][1] для OCamlза исключением того, что Myriad не интегрирован такжу, как и OCaml. Хотя интеграция в F# компилятор и желанна для такого инструмента, вопрос в том, будет ли он в возможностях для будущего направления языка F#. Здесь также существует фактор задержки написания, расширения для компилятора, и ожидания релиза, чтобы он стал общедоступным для всех для использования. Усиливая MSBuild, становится возможным приблизиться к некоторым возможностям и функционалу [ppx_deriving][1]. Как минимум он развивает идею того, что более продвинутые макро возможности могли бы выглядеть.

[ppx_deriving][1] работает, позволяя типу расширяться с помощью произвольной функции, существуют множество встроенных плагинов таких как, `show`, `eq`, `ord`, `enum`, `iter`, `map`, `fold`, `make`, `yojson` and `protobuf`.   

Другой хорошо знакомый плагин это [ppx_fields_conv][2]:
>Поколение средств доступа и итерационных функций для записей OCaml.
ppx_fields_conv это ppx переписчик, который может быть использован, чтобы определить значения первого класса представляющих поля записи, и дополнительные процедуры, для установки и получения полей записи, повторения и складывания всех полей записи и создания новых значений рекордов.  

Это была основная идея Myriad. Мы не используем полную возможность [ppx_fields_conv][2], только подмножество, чтобы показать потенциал этого подхода. Более специфично мы сделаем функцию средства доступа к полям и сделаем функцию для каждого рекорда во входных данных.

## Использование и Демо

### Входящий код

[Myriad][17] работает с входящем файлом как с основой для кодогенерации, в частности записи во входящем файле используются в фазе генерации кода.


```fsharp
namespace Example

type Test1 = { one: int; two: string; three: float; four: float32 }
type Test2 = { one: Test1; two: string }
```

[Myriad][17] is invoked via the following addition to an F# project file:

```
<Compile Include="Generated.fs" > <!--1-->
    <MyriadFile>..\..\src\Example\Library.fs</MyriadFile> <!--2-->
    <MyriadNameSpace>Test</MyriadNameSpace> <!--3-->
</Compile>
```
1. Элемент `<Compile Include="..."` используется, чтобы задать исходящее название и так же, чтобы убедиться, что сгенерированный код используется вовремя компиляции.
2. `<MyriadFile>...` используется, чтобы выбрать файл в качестве входящих данных в кодогенерацию [Myriad][17].  
3. `<MyriadNameSpace>...` используется, чтобы задать пространство имен, используемое в сгенерированном коде. Если его пропустить, то будет использован `RootNamespace` из файла проекта.

### Исходящий код

```fsharp
//------------------------------------------------------------------------------
//        This code was generated by myriad.
//        Changes to this file will be lost when the code is regenerated.
//------------------------------------------------------------------------------
namespace rec Test

module Test1 =
    open Example

    let one (x : Test1) = x.one
    let two (x : Test1) = x.two
    let three (x : Test1) = x.three
    let four (x : Test1) = x.four

    let create (one : int) (two : string) (three : float) (four : float32) : Test1 =
        { one = one
          two = two
          three = three
          four = four }

module Test2 =
    open Example

    let one (x : Test2) = x.one
    let two (x : Test2) = x.two

    let create (one : Test1) (two : string) : Test2 =
        { one = one
          two = two }
```

Технические аспекты этого проекта исходят из четырех основных областей: парсинг, построение АСТ, исходящий код и интеграция сборки.

## Парсинг

Первый шаг это собрать информацию из входящего файла, что можно легко сделать используя [FSharp.Compiler.Services][3]. Довольно легко извлечь АСТ из куска кода, используя что-то вроде этого:


```fsharp
let filename = "test.fs"
let fileText = File.ReadAllText filename
let checker = FSharpChecker.Create()
let projOptions, _ = checker.GetProjectOptionsFromScript(filename, fileText) |> Async.RunSynchronously

let ast =
    let parsingOptions, _ = checker.GetParsingOptionsFromProjectOptions(projOptions)
    let parseFileResults = checker.ParseFile(file, input, parsingOptions) |> Async.RunSynchronously
    match parseFileResults.ParseTree with
    | Some tree -> tree
    | None -> failwith "Something went wrong during parsing!"
```

`Сhecker` создается и `projectOptions` создаются, используя `filename` и `fileText` как входящие данные.
Теперь 'cheker' может быть использован, чтобы извлечь АСТ, сначала создав `parsingOptions`, и передав их на `checker.ParseFile`. Эта функция возвращает параметр, который мы 
сопоставляем с шаблонами, выбрасывая исключение, если АСТ не представлено. 

Теперь, когда мы имеем АСТ, мы можем чаще использовать сопоставление шаблонов, чтобы попытаться и найти заметки АСТ, в которых мы заинтересованы. Это может быть сделано, используя небольшую секцию сложного сопоставления шаблонов и декомпозицию:

```fsharp
match ast with
| ParsedInput.ImplFile(ParsedImplFileInput(_,_,_,_,_,modules,_)) ->
    for SynModuleOrNamespace(namespaceIdent,_,_,moduleDecls,_,_,_,_) in modules do
        for moduleDecl in moduleDecls do
            match moduleDecl with
            | SynModuleDecl.Types(types,_) ->
                for TypeDefn(ComponentInfo(_,_,_,recordIdent,_,_,_,_), typeDefRepr,_,_) in types do
                    match typeDefRepr with
                    | SynTypeDefnRepr.Simple(SynTypeDefnSimpleRepr.Record(_,fields,_),_) ->
                        yield (namespaceIdent,recordIdent,fields)
//...
```
В этом фрагменте вы можете увидеть, что мы проходим через первый декомпозирующий АСТ метод `ParsedInput.ImplFile`. Мы делаем это, чтобы нам не приходилось извлекать дальнейшую информацию в другом сопоставлении таком как:


```fsharp
match ast with
| ParsedInput.ImplFile(pu) ->
    match pu with ParsedImplFileInput(_,_,_,_,_,modules,_)
```

Сейчас мы можем петлять через модули и пространства имен. Затем мы забуриваемся глубже, пока не найдем объявления типов в модуле. Как только мы нашли объявление типа, мы можем
сопоставить по узлу записи через `SynTypeDefnSimpleRepr.Record`. После того, как мы нашли запись, мы можем извлечь и yield параметры, которые нам нужны для генератора. В этом примере все, что нам нужно - это родительское пространство имен, которое мы можем найти в SynModuleOrNamespace(namespaceIdent,_,_,_,_,_,_,_)`, идентификатор записи, который мы можем найти в объявление типа 'ComponentInfo': `TypeDefn(ComponentInfo(_,_,_,recordIdent,_,_,_,_),_,_,_)`. Последний параметр, который нам нужен - это поля записи, которые находятся в самом определении записи:`SynTypeDefnSimpleRepr.Record(_,fields,_)`. В этой секции вы можете увидеть, что мы активно использовали сопоставление шаблонов и декомпозицию размеченных объединений, что идеально для этой конкретной задачи.

## Построение  Ast 
Так как мы знаемнужную нам информацию, мы теперь можем начать конструировать модули и функции, которые мы хотим сгенерировать. 

Для построения АСТ мы воспользуемся библиотекой [FSAst][16].  Я так же использовал ее в Falanx, но решил объяснить это более детально здесь.  

Принцип работы [FsAst][16] в том, что библиотека обертывает простые узлы АСТ в типы записей, которые имеют значения по умолчанию, что позволит нам использовать синтаксис обновления записи.

Это пример `Let x = 42`:

Let это узел типа `SynModuleDecl`, вот его объявление:

```fsharp
SynModuleDecl =
| ModuleAbbrev of ident: Ident * longId: LongIdent * range: range
| NestedModule of SynComponentInfo * isRecursive: bool * SynModuleDecls * bool * range: range
| Let of isRecursive: bool * SynBinding list * range: range
| DoExpr of SequencePointInfoForBinding * SynExpr * range: range
| Types of SynTypeDefn list * range: range
| Exception of SynExceptionDefn * range: range
| Open of longDotId: LongIdentWithDots * range: range
| Attributes of SynAttributes * range: range
| HashDirective of ParsedHashDirective * range: range
| NamespaceFragment of SynModuleOrNamespace
```

Одна привязка `Let x = 42` выглядит так:
```
Let
   (false,
    [Binding
       (None,NormalBinding,false,false,[],
        PreXmlDoc ((2,5),Microsoft.FSharp.Compiler.Ast+XmlDocCollector),
        SynValData (None,SynValInfo ([],SynArgInfo ([],false,None)),None),
        Named
          (Wild tmp.fsx (2,4--2,5) IsSynthetic=false,x,false,None,
           tmp.fsx (2,4--2,5) IsSynthetic=false),None,
        Const (Int32 42,tmp.fsx (2,8--2,10) IsSynthetic=false),
        tmp.fsx (2,4--2,5) IsSynthetic=false,
        SequencePointAtBinding tmp.fsx (2,0--2,10) IsSynthetic=false)],
    tmp.fsx (2,0--2,10) IsSynthetic=false)
```

Само по себе оно бы объявлялось как `SynModuleDecl.Let(false, bindings, range)`, что потребовало бы построение списка привязок. Привязка определяется тследущим образом:

```fsharp
SynBinding =
| Binding of
    accessibility: SynAccess option *
    kind: SynBindingKind *
    mustInline: bool *
    isMutable: bool *
    attrs: SynAttributes *
    xmlDoc: PreXmlDoc *
    valData: SynValData *
    headPat: SynPat *
    returnInfo: SynBindingReturnInfo option *
    expr: SynExpr  *
    range: range *
    seqPoint: SequencePointInfoForBinding
```

```
SynBinding.Binding(
    None,
    NormalBinding,
    false,
    false,
    [],
    PreXmlDoc(range.zero, XmlDocCollector()),
    SynValData(None, SynValInfo([], SynArgInfo ([], false, None)), None),
    Named(Wild, "tmp.fsx, range.zero, IsSynthetic=false, "x", false, None, "tmp.fsx", range.zero, IsSynthetic=false),
    None,
    Const (Int32, 42,"tmp.fsx", range.zero, IsSynthetic=false),
    "tmp.fsx", range.zero, IsSynthetic=false,
    SequencePointAtBinding("tmp.fsx", range.zero, IsSynthetic=false)
```

Вы можете увидеть, что объявление этих узлов может стать очень сложным довольно быстро. С [FsAst][16] мы можем определять упомянутый выше фрагмент АСТ Let подобно этому: 

```fsharp
SynModuleDecl.CreateLet(
    { SynBindingRcd.Let with
        Pattern = SynPatRcd.CreateNamed(Ident.Create "x", SynPatRcd.CreateWild)
        Expr = SynExpr.CreateConst(SynConst.Int32 42)
    } )
```
Это делает построение фрагментов АСТ гораздно легче!

Этот отрывок кода из [Myriad][17], который создает маппинг поля записи:
```fsharp
let createMap (parent: LongIdent) (field: SynField)  =
    let field = field.ToRcd
    let fieldName = match field.Id with None -> failwith "no field name" | Some f -> f 

    let recordType =
        LongIdentWithDots.Create (parent |> List.map (fun i -> i.idText))
        |> SynType.CreateLongIdent

    let varName = "x"
    let pattern =
        let name = LongIdentWithDots.Create([fieldName.idText])
        let arg =
            let named = SynPatRcd.CreateNamed(Ident.Create varName, SynPatRcd.CreateWild )
            SynPatRcd.CreateTyped(named, recordType)
            |> SynPatRcd.CreateParen

        SynPatRcd.CreateLongIdent(name, [arg])

    let expr =
        let ident = LongIdentWithDots.Create [ yield varName; yield fieldName.idText]
        SynExpr.CreateLongIdent(false, ident, None)

    let valData =
        let argInfo = SynArgInfo.CreateIdString "x"
        let valInfo = SynValInfo.SynValInfo([[argInfo]], SynArgInfo.Empty)
        SynValData.SynValData(None, valInfo, None)

    SynModuleDecl.CreateLet [{SynBindingRcd.Let with
                                Pattern = pattern
                                Expr = expr
                                ValData = valData }]
```

Позвольте мне пробежаться по разделам, которые составляют привязку Let:
```fsharp
let one (x : Test1) = x.one
```

#### Pattern

`Pattern` это имя привязки, в данном случае: `one (x : Test1)`
Мы создаем `LongIdentifier` из fieldNames.idText, так как имя привязки Let будет то же, что и имя поля, затем мы делаем аргумент, используя `"x"` как `varName`

#### Expr
`Expr` это выражение, которое вы привязываете к имени, то есть это: `x.one`, что по существу всего лишь `LongIdentWithDots` от
 `varName` `.` `fieldName`.  

#### ValData
`ValData` это информация об именах аргументов и других метаданных для параметра member function. Таких как опциональные параметры или любые свойства, атрибуты, применяемые к параметрам.  В этом примере мы просто добавили идентификатор к аргументу. 

Использование FSAst значительно упрощает создание F# AST с помощью кода, но все еще есть много улучшений, которые могли бы быть сделаны с созданием 'Ident' и других областей, возможно существует множество фукций внутри компилятора, которые могли бы нам помочь с этим.  

## Исходящий код  
Реальная ккодогенерация может быть сделана, используя F# инструмент форматирования [Fantomas][4].  [Fantomas][4] есть вызов API, который принимает AST и форматирует код, который оно представляет. Все что нам нужно сделать, это бросить вызов API, чтобы получить обратно отформатированный исходный код и добавляет его в заголовок:

```fsharp
let sourceCode = Fantomas.CodeFormatter.FormatAST(ast, filename, None, fantomasConfig)
```

Теперь можно вставить заголовок и записать его в файл:
```fsharp
let code =
    [   "//------------------------------------------------------------------------------"
        "//        This code was generated by myriad."
        "//        Changes to this file will be lost when the code is regenerated."
        "//------------------------------------------------------------------------------"
        formattedCode ]
    |> String.concat Environment.NewLine

File.WriteAllText(outputFile, code)
```

## Интеграция MSBuild
Обертывание парсинга и кодогенерации в меннере легкой для использования, делается с помощью расширения MSBuild, это дает приближение к использованию [ppx_deriving][1] и его роль внутри экосистеме OCaml.

Это достигается добавлением двух дочерних атрибьютов элементу MSBuild 'Compile' следующим образом:

```
<Compile Include="Generated.fs">
    <MyriadFile>..\..\src\Example\Library.fs</MyriadFile>
    <MyriadNameSpace>Test</MyriadNameSpace>
</Compile>
```
Элемент `<Compile Include="Generated.fs" >` используется, чтобы указать имя исходящего файла, а так же, чтобы убедиться, что сгенерированный используется во время компиляции.  

`<MyriadFile>..\..\src\Example\Library.fs</MyriadFile>` используется, чтобы выбрать файл как входящие данные для кодогенерации.  

`<MyriadNameSpace>Test</MyriadNameSpace>` используется, чтобы указать пространство имен для использования в сгенерированном коде.  Если это опустить, то `RootNamespace` будет использовано. 

### Целевой объект _MyriadSdkFilesList

Чтобы интеграция произошла, `MyriadFile` и `MyriadNameSpace` должны быть обработаны расширением MSBUILd и сформировать список элементных расширений 'Compile', который мы можем использовать для формирования входящих данных в CLI/Command line tool. Это делается `_MyriadSdkFilesList` Target, первая часть продемонстрирована дальше:   
```
<Target Name="_MyriadSdkFilesList" BeforeTargets="MyriadSdkGenerateInputCache">
    <ItemGroup>
        <MyriadSource Include="%(Compile.MyriadFile)" Condition=" '%(Compile.MyriadFile)' != '' ">
            <OutputPath>$([System.IO.Path]::GetFullPath('%(Compile.FullPath)'))</OutputPath>
            <Namespace Condition=" '%(Compile.MyriadNamespace)' != '' " >%(Compile.MyriadNamespace)</Namespace>
            <Namespace Condition=" '%(Compile.MyriadNamespace)' == '' " >$(RootNamespace)</Namespace>
        </MyriadSource>
    </ItemGroup>

    <ItemGroup>
        <MyriadCodegen Include="%(MyriadSource.FullPath)">
            <OutputPath Condition=" '%(MyriadSource.OutputPath)' != '' ">$([System.IO.Path]::GetFullPath('%(MyriadSource.OutputPath)'))</OutputPath>
            <OutputPath Condition=" '%(MyriadSource.OutputPath)' == '' ">%(MyriadSource.FullPath).fs</OutputPath>
            <Namespace>%(MyriadSource.Namespace)</Namespace>
        </MyriadCodegen>
    </ItemGroup>

    <PropertyGroup>
        <_MyriadSdkCodeGenInputCache>$(IntermediateOutputPath)$(MSBuildProjectFile).FalanxSdkCodeGenInputs.cache</_MyriadSdkCodeGenInputCache>
    </PropertyGroup>
</Target>
```
Сначала мы собираем список файлов для обработки [Myriad'ом][17].  Мы cделаем это, создав 'ItemGroup', которая является списком элементов 'MyriadSource', только 'Compile' элементы, имеющие узлы 'MyriadFile', обрабатываются, это обеспечивается атрибутом `Condition`: `Condition=" '%(Compile.MyriadFile)' != ''`.  Элемент `MyriadSource` формируется из трех кусков информации.  

Атрибут `Include` это элемент `MyriadFile`, который мы включаем в файл MSBuild.  
Элемент `OutputPath` это полный путь для атрибута `Include` элементов `Compile`, также изветсный как `Identity`
Элемент `Namespace` это или `%(Compile.MyriadNamespace)`, если оно представлено или `$(RootNamespace)`, если нет.   
- - -
Сейчас, когда мы создали 'ItemGroup' , содержащий элементы 'MyriadSource', мы можем немного переделать ее, вы можете свернуть эти изменения в `MyriadSource` `ItemGroup`, но гораздно легче создать две 'ItemGroup'.

Мы создаем новую 'ItemGroup' под названием 'MyriadCodegen', которая ссылается на 'MyriadSource' ради его атрибута 'Include'. Так же существуют атрибуты 'Condition' для проверки того, что 'OutputPath' не пустой и для того, что бы убедиться, что если он пустой, нужно просто установить его во входящий файл. Это будет означать, что входящий файл сам будет обработан и изменен, а не будет вписан в другой файл.

### Целевой объект MyriadSdkGenerateCode

Финальный шаг состоит в том, чтобы вызвать инструмент CLI со всей информацией, которую мы собрали в 'MyriadSdkGenerateCode' целевой объект:
```
<PropertyGroup>
    <MyriadSdkGenerateCodeDependsOn>$(MyriadSdkGenerateCodeDependsOn);ResolveReferences;MyriadSdkGenerateInputCache</MyriadSdkGenerateCodeDependsOn>
</PropertyGroup>

<Target Name="MyriadSdkGenerateCode"
        DependsOnTargets="$(MyriadSdkGenerateCodeDependsOn)" 
        BeforeTargets="CoreCompile"
        Condition=" '$(DesignTimeBuild)' != 'true' "
        Inputs="@(MyriadCodegen);$(_MyriadSdkCodeGenInputCache);$(MyriadSdk_Generator_Exe)"
        Outputs="%(MyriadCodegen.OutputPath)">

    <PropertyGroup>
        <_MyriadSdk_InputFileName>%(MyriadCodegen.Identity)</_MyriadSdk_InputFileName>
        <_MyriadSdk_OutputFileName>%(MyriadCodegen.OutputPath)</_MyriadSdk_OutputFileName>
        <_MyriadSdk_Namespace>%(MyriadCodegen.Namespace)</_MyriadSdk_Namespace>
    </PropertyGroup>

    <ItemGroup>
        <MyriadSdk_Args Include='--inputfile "$(_MyriadSdk_InputFileName)"' />
        <MyriadSdk_Args Include='--outputfile "$(_MyriadSdk_OutputFileName)"' />
        <MyriadSdk_Args Include='--namespace "$(_MyriadSdk_Namespace)"' />
    </ItemGroup>

    <!-- Use dotnet to execute the process. -->
    <Exec Command="$(MyriadSdk_Generator_ExeHost)&quot;$(MyriadSdk_Generator_Exe)&quot; @(MyriadSdk_Args -> '%(Identity)', ' ')" />
</Target>
```

Атрибут 'DependsOnTarget' используется, чтобы гарантировать, что все, что содержится в элементе 'MyriadSdkGenerateCodeDependsOn' запущен до этого целевой объект.

Внутри элемента 'Target' `MyriadSdkGenerateCode`, есть атрибутs 'Inputs' и 'Outputs', они используються, чтобы обозначить, когда [Myriad][17] нужно запуститься. Предмет считается up-to-date, если его исходящий файл настолько же старый или новее, чем его входящий файл/файлы.

Мы создаем  'PropertyGroup', чтобы содержать параметры командной строки `_MyriadSdk_InputFileName`, `_MyriadSdk_OutputFileName` и `_MyriadSdk_Namespace` используя соответствующие элементы из ItemGroup `MyriadCodegen` `_MyriadSdkFilesList` целевой группы.

После того мы создаем 'ItemGroup', которая имеет внутри себя три элемента `MyriadSdk_Args`, с помощью которых мы должны вызвать генератор кода.

В финале мы запускаем кодогенератор с 'Exec', так же вызывая CLI, с параметрами из атрибута `MyriadSdk_Args` `Include` с помощью функции MSBuild `@(MyriadSdk_Args -> '%(Identity)', ' ')`

Единственная вещь, не обсужденная нами в этом разделе, был  `_MyriadSdkCodeGenInputCache`, упомянутый выше в обоих целевых объектах:
```
<Target Name="MyriadSdkGenerateInputCache" DependsOnTargets="ResolveAssemblyReferences;_MyriadSdkFilesList" BeforeTargets="MyriadSdkGenerateCode">

    <ItemGroup>
        <MyriadSdk_CodeGenInputs Include="@(MyriadCodegen);@(ReferencePath);$(MyriadSdk_Generator_Exe)" />
    </ItemGroup>

    <Hash ItemsToHash="@(MyriadSdk_CodeGenInputs)">
        <Output TaskParameter="HashResult" PropertyName="MyriadSdk_UpdatedInputCacheContents" />
    </Hash>

    <WriteLinesToFile Overwrite="true" File="$(_MyriadSdkCodeGenInputCache)" Lines="$(MyriadSdk_UpdatedInputCacheContents)" WriteOnlyWhenDifferent="True" />

</Target>
```

Этот целевой объект генерирует хэш, используя `@(MyriadCodegen);@(ReferencePath);$(MyriadSdk_Generator_Exe)` как входящие данные. Если какое-нибудь из них изменить, тогда другой хэш будет прописан в исходящем файле через `<WriteLinesToFile Overwrite="true" File="$(_MyriadSdkCodeGenInputCache)"`. Он охватывает полный набор всех входящих данных в кодогенератор. Основывается это на целевом объекте _GenerateCompileDependencyCache из .NET project system, который использовается как ссылка. Вы можете найти его в [источнике][5] .Net project system. 

# Итог

Вау, это заняло немного больше чем, я думал, я надеюсь, что я не слишком много болтаю. Это довольно большая тема, как только вы попробуете разобраться в деталях. Возможно я мог бы написать множество сочинений на каждую тему. 

Я думаю, что ключевым аспектам, который я суммировал в разделе по [Falanx][6] , был в том, что с цитированием довольно сложно работать, и что это действительно так, если вы пытаетесь разработать их до предела. [Type Providers][9] и аппликативное программирование с множеством универсадльных параметров уж точно!

Фаланкс был большим проектом, он показал, что даже не входящие в стандартный набор библиотеки могут быть использованы для создания довольно сложного проекта. Создание цитирования и из литералов, и из выражений занимает много времени и зачастую вам нужно  провести множество отладок, чтобы сделать все правильно. [Myriad][17] строится на этом, предоставляя более простые решения составляя АСТ напрямую. Я думаю есть небольшая возможность для компромисса между ними, где литералы цитирования могут быть использованы как ярлык для предоставления фрагмента АСТ, в том виде, в котором это делается в [Haxe][19] [Expression Reification][20].  Это делается тем же образом, каким вы можете использовать цитирование для извлечения `MethodInfo`.   (https://github.com/7sharp9/falanx/blob/master/src/Falanx.Machinery/Expr.fs#L199):
```fsharp
<@@ writeEmbedded x x x @@>
|> Expr.methodof
|> Expr.callStatic [position; buffer; value]
```
Я думаю некоторые креативные применения перехода к или от цитирования, и АСТ позволит немного больше гибкости в метапрограммировании.

Вы можете найти репозитории для [Myriad][17] и [Falanx][6] на github, как только Ive опубликует этот пост, они также доступны в Nuget: [Falanx.Sdk](https://www.nuget.org/packages/Falanx.Sdk/)/[Myriad.Sdk](https://www.nuget.org/packages/Myriad.Sdk/)

Я надеюсь, что вам понравился данный пост по применяемому метапрограммированию, и углубиться в этот предмет и многие другие на моем [YouTube канале][18].  

До следующего раза! 

# References
<small>
[1 Type-driven code generation for OCaml - ppx_deriving][1]  
[2 Generation of accessor and iteration functions for OCaml records][2]  
[3 Compiler Services: Processing untyped syntax tree][3]  
[4 Fantomas F# source code formatter][4]  
[5 .NET project System:- _GenerateCompileDependencyCache][5]  
[6 Falanx protobuf Code Generation][6]  
[7 Meta-matic][7]  
[8 Type Provider SDK][8]  
[9 Type Providers][9]  
[10 Protocol Buffers][10]  
[11 Fleece][11]  
[12 Froto][12]  
[13 Quotation Compiler][13]  
[14 Statically Resolved Type Parameters][14]  
[15 Quotations][15]  
[16 FsAst][16]  
[17 Myriad][17]  
[18 My YouTube Channel][18]  
[19 Haxe][19]  
[20 Expression Reification][20]  
[21 CLI Tool][21]  
[22 Discriminated Unions][22]  
[23 Records][23]
</small>

[1]: https://github.com/ocaml-ppx/ppx_deriving
[2]: https://github.com/janestreet/ppx_fields_conv
[3]: https://fsharp.github.io/FSharp.Compiler.Service/untypedtree.html
[4]: https://github.com/fsprojects/fantomas
[5]: https://github.com/Microsoft/msbuild/blob/adb180d394176f36aca1cc2eac4455fef564739f/src/Tasks/Microsoft.Common.CurrentVersion.targets#L3407
[6]: https://github.com/7sharp9/falanx
[7]: https://7sharp9.github.io/2015/07/12/2015-07-08-meta-matic/
[8]: https://github.com/fsprojects/FSharp.TypeProviders.SDK
[9]: https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/type-providers/
[10]: https://developers.google.com/protocol-buffers/
[11]: https://github.com/mausch/Fleece
[12]: https://github.com/ctaggart/froto
[13]: https://github.com/eiriktsarpalis/QuotationCompiler
[14]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/generics/statically-resolved-type-parameters
[15]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/code-quotations
[16]: https://github.com/ctaggart/FsAst
[17]: https://github.com/7sharp9/myriad
[18]: https://www.youtube.com/channel/UC0kXc1f_WBYSklrElcPWzgg
[19]: https://haxe.org
[20]: https://haxe.org/manual/macro-reification-expression.html
[21]: https://docs.microsoft.com/en-us/dotnet/core/tools/?tabs=netcore2x
[22]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions
[23]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/records