# json-query-2-javascript

## Zespół
- Dzmitry Nikitsin — dnikitin@student.agh.edu.pl
- Niaz Lapkouski

## Założenia programu
Translator zapytań SQL-like dla danych zapisanych w formacie JSON.  
Użytkownik podaje zapytanie w uproszczonym DSL inspirowanym SQL oraz plik/zmienną JSON, a program wykonuje pełny pipeline kompilatorski: lekser → parser → AST → analiza semantyczna → generacja kodu JavaScript.

## Ogólne cele programu
Program umożliwia wykonywanie zapytań na danych JSON poprzez tłumaczenie ich do kodu JavaScript.

## Rodzaj translatora
Kompilator (translator do kodu docelowego JS).

## Pipeline
```
zapytanie DSL
    │
    ▼
 Lexer (ANTLR4)          ← tokeny
    │
    ▼
 Parser (ANTLR4)         ← drzewo rozbioru (Parse Tree)
    │
    ▼
 AST Visitor             ← własne węzły AST
    │
    ▼
 Semantic Analyzer       ← sprawdzanie ścieżek, typów, UNNEST
    │
    ▼
 Code Generator          ← kod JavaScript
    │
    ▼
 Wykonanie JS na danych JSON → wynik (JSON / terminal)
```

## Planowany wynik działania programu
Generator kodu JavaScript realizującego zapytanie na pliku JSON.

## Planowany język implementacji
JavaScript.

## Generator parsera
ANTLR4.

---

## Kluczowe decyzje semantyczne

| Kwestia | Decyzja |
|---|---|
| Nawigacja przez zagnieżdżone **obiekty** | Dozwolona bezpośrednio: `address.city` |
| Nawigacja przez **tablice** | Wymaga jawnego `UNNEST(pole) AS alias` |
| Agregatów | Tylko `COUNT(ścieżka)` — zlicza elementy tablicy danego rekordu |
| Joinów | Brak pełnych join-ów; dwa pliki JSON można łączyć przez proste UNNEST |
| Wynik | Wewnętrznie lista obiektów JS; eksport jako JSON lub wydruk do terminala |

---

## Wspierane zapytania — przykłady

```sql
-- Prosty filtr z zagnieżdżoną ścieżką
SELECT name, address.city FROM users WHERE age > 18 ORDER BY name ASC LIMIT 10

-- Rozwinięcie tablicy przez UNNEST
SELECT name, tag FROM users UNNEST(tags) AS tag WHERE tag = 'admin'

-- Zliczanie elementów tablicy (agregat na poziomie rekordu)
SELECT name, COUNT(orders) FROM customers WHERE COUNT(orders) > 3

-- Złożone warunki i sortowanie po głębokiej ścieżce
SELECT id, profile.bio FROM users
WHERE age >= 21 AND profile.active = true
ORDER BY profile.score DESC
LIMIT 5
```

---

## Opis tokenów

Tokeny są podzielone na cztery kategorie: **słowa kluczowe**, **literały**, **identyfikatory** i **operatory/znaki przestankowe**.  
Wszystkie słowa kluczowe są **case-insensitive** (obsługa na poziomie leksera przez klasy znaków ANTLR4).  
Białe znaki i komentarze liniowe (`--`) są pomijane (nie trafiają do strumienia tokenów).

### Słowa kluczowe

| Token | Wzorzec (regex) | Opis |
|---|---|---|
| `SELECT` | `[Ss][Ee][Ll][Ee][Cc][Tt]` | Rozpoczyna listę projekcji |
| `FROM` | `[Ff][Rr][Oo][Mm]` | Określa źródło danych |
| `WHERE` | `[Ww][Hh][Ee][Rr][Ee]` | Filtrowanie wierszy |
| `ORDER` | `[Oo][Rr][Dd][Ee][Rr]` | Sortowanie (razem z `BY`) |
| `BY` | `[Bb][Yy]` | Część `ORDER BY` |
| `LIMIT` | `[Ll][Ii][Mm][Ii][Tt]` | Ograniczenie liczby wyników |
| `UNNEST` | `[Uu][Nn][Nn][Ee][Ss][Tt]` | Rozwinięcie tablicy do wierszy |
| `AS` | `[Aa][Ss]` | Alias wyrażenia lub rozwinięcia |
| `AND` | `[Aa][Nn][Dd]` | Koniunkcja logiczna |
| `OR` | `[Oo][Rr]` | Alternatywa logiczna |
| `NOT` | `[Nn][Oo][Tt]` | Negacja logiczna (prefiks) |
| `ASC` | `[Aa][Ss][Cc]` | Sortowanie rosnące |
| `DESC` | `[Dd][Ee][Ss][Cc]` | Sortowanie malejące |
| `COUNT` | `[Cc][Oo][Uu][Nn][Tt]` | Agregat: liczba elementów tablicy |
| `NULL` | `[Nn][Uu][Ll][Ll]` | Literał null |

### Literały

| Token | Wzorzec (regex) | Przykłady | Opis |
|---|---|---|---|
| `BOOLEAN_LIT` | `true\|false` (case-insensitive) | `true`, `False`, `TRUE` | Wartość logiczna |
| `INTEGER_LIT` | `[0-9]+` | `0`, `42`, `1000` | Liczba całkowita bez znaku |
| `FLOAT_LIT` | `[0-9]+'.'[0-9]*` lub `'.'[0-9]+` | `3.14`, `.5`, `2.` | Liczba zmiennoprzecinkowa bez znaku |
| `STRING_LIT` | `"..."` lub `'...'` z obsługą `\`-escape | `"hello"`, `'world'`, `"O\'Brien"` | Łańcuch znaków |

### Identyfikatory

| Token | Wzorzec (regex) | Opis |
|---|---|---|
| `IDENTIFIER` | `[a-zA-Z_][a-zA-Z_0-9]*` | Nazwa zmiennej, pola, aliasu. Słowa kluczowe mają pierwszeństwo przy identycznym dopasowaniu. |

### Operatory i znaki przestankowe

| Token | Leksem | Opis |
|---|---|---|
| `EQ` | `=` | Równość |
| `NEQ` | `!=` | Nierówność |
| `LT` | `<` | Mniejszy niż |
| `GT` | `>` | Większy niż |
| `LEQ` | `<=` | Mniejszy lub równy |
| `GEQ` | `>=` | Większy lub równy |
| `PLUS` | `+` | Dodawanie |
| `MINUS` | `-` | Odejmowanie / negacja unaryczna |
| `STAR` | `*` | Mnożenie / SELECT * |
| `SLASH` | `/` | Dzielenie |
| `LPAREN` | `(` | Nawias otwierający |
| `RPAREN` | `)` | Nawias zamykający |
| `COMMA` | `,` | Separator elementów listy |
| `DOT` | `.` | Separator segmentów ścieżki |

### Tokeny pomijane

| Token | Wzorzec | Opis |
|---|---|---|
| `WS` | `[ \t\r\n]+` | Białe znaki — pomijane |
| `LINE_COMMENT` | `'--' [^\r\n]*` | Komentarz liniowy — pomijany |

---

## Gramatyka w notacji ANTLR4

Pełny plik gramatyki: [grammar/JsonQuery.g4](grammar/JsonQuery.g4)

Poniżej opis struktury gramatyki z komentarzem do kluczowych decyzji.

### Reguły parsera

```antlr
grammar JsonQuery;

query
    : selectStmt EOF
    ;

selectStmt
    : SELECT selectList
      FROM source
      unnestClause*
      whereClause?
      orderByClause?
      limitClause?
    ;

selectList
    : STAR                                  # SelectAll
    | selectItem (COMMA selectItem)*        # SelectItems
    ;

selectItem
    : expr (AS IDENTIFIER)?
    ;

source
    : IDENTIFIER
    ;

unnestClause
    : UNNEST LPAREN path RPAREN AS IDENTIFIER
    ;

whereClause
    : WHERE expr
    ;

orderByClause
    : ORDER BY orderItem (COMMA orderItem)*
    ;

orderItem
    : path direction=(ASC | DESC)?
    ;

limitClause
    : LIMIT INTEGER_LIT
    ;

// Priorytety operatorów zakodowane kolejnością alternatyw
// (pierwsza = najniższy priorytet, ostatnia = najwyższy)
expr
    : expr OR  expr                         # OrExpr
    | expr AND expr                         # AndExpr
    | NOT expr                              # NotExpr
    | expr compOp expr                      # CompareExpr
    | expr (PLUS  | MINUS) expr             # AddExpr
    | expr (STAR  | SLASH) expr             # MulExpr
    | MINUS expr                            # UnaryMinus
    | primary                               # PrimaryExpr
    ;

primary
    : COUNT LPAREN path RPAREN              # CountAgg
    | path                                  # PathExpr
    | literal                               # LiteralExpr
    | LPAREN expr RPAREN                    # ParenExpr
    ;

path
    : IDENTIFIER (DOT IDENTIFIER)*
    ;

literal
    : INTEGER_LIT
    | FLOAT_LIT
    | STRING_LIT
    | BOOLEAN_LIT
    | NULL
    ;

compOp
    : EQ | NEQ | LT | GT | LEQ | GEQ
    ;
```

### Reguły leksera

```antlr
// Słowa kluczowe (case-insensitive, przed IDENTIFIER)
SELECT  : [Ss][Ee][Ll][Ee][Cc][Tt] ;
FROM    : [Ff][Rr][Oo][Mm] ;
WHERE   : [Ww][Hh][Ee][Rr][Ee] ;
ORDER   : [Oo][Rr][Dd][Ee][Rr] ;
BY      : [Bb][Yy] ;
LIMIT   : [Ll][Ii][Mm][Ii][Tt] ;
UNNEST  : [Uu][Nn][Nn][Ee][Ss][Tt] ;
AS      : [Aa][Ss] ;
AND     : [Aa][Nn][Dd] ;
OR      : [Oo][Rr] ;
NOT     : [Nn][Oo][Tt] ;
ASC     : [Aa][Ss][Cc] ;
DESC    : [Dd][Ee][Ss][Cc] ;
COUNT   : [Cc][Oo][Uu][Nn][Tt] ;
NULL    : [Nn][Uu][Ll][Ll] ;

BOOLEAN_LIT : [Tt][Rr][Uu][Ee] | [Ff][Aa][Ll][Ss][Ee] ;

IDENTIFIER  : [a-zA-Z_] [a-zA-Z_0-9]* ;

INTEGER_LIT : [0-9]+ ;
FLOAT_LIT   : [0-9]+ '.' [0-9]* | '.' [0-9]+ ;
STRING_LIT  : '"'  ( ~["\\\r\n] | '\\' . )* '"'
            | '\'' ( ~['\\\r\n] | '\\' . )* '\''
            ;

// Operatory (wieloznakowe przed jednoznakowymi)
STAR    : '*' ;   COMMA   : ',' ;   DOT     : '.' ;
LPAREN  : '(' ;   RPAREN  : ')' ;
PLUS    : '+' ;   MINUS   : '-' ;   SLASH   : '/' ;
LEQ     : '<=' ;  GEQ     : '>=' ;  NEQ     : '!=' ;
EQ      : '='  ;  LT      : '<'  ;  GT      : '>'  ;

// Pomijane
WS          : [ \t\r\n]+  -> skip ;
LINE_COMMENT : '--' ~[\r\n]* -> skip ;
```

### Priorytety i łączność operatorów (podsumowanie)

| Poziom | Operatory | Łączność |
|---|---|---|
| 1 (najniższy) | `OR` | lewostronna |
| 2 | `AND` | lewostronna |
| 3 | `NOT` | prawostronna (prefiks) |
| 4 | `=` `!=` `<` `>` `<=` `>=` | brak łączności (nieporównywalne) |
| 5 | `+` `-` | lewostronna |
| 6 | `*` `/` | lewostronna |
| 7 | unarny `-` | prawostronna (prefiks) |
| 8 (najwyższy) | `COUNT(...)`, ścieżka, literał, `(...)` | — |

