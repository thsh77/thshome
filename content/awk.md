---
title: "Awk"
date: 2020-12-14T19:53:50+02:00
draft: true
---

Et awk-script består af et mønster (_pattern_) og en handling
(_action_). Et mønster kan være:

1. Regulære udtryk: `/regex/`
2. Relationsudtryk, som gør brug af diverse operatører til
   tilskrivning (`=`, ), logiske udtryk (`||`, `&&` ), matchende
   regulære udtryk (`~`, `!~`) osv.
3. `BEGIN`- eller `END`-mønster
4. Rækkeviddemønster: `/regex/,/regex/`

En handling er et eller flere udsagn, som udføres på de inputlinjer,
som matcher mønstret. Hvis ikke der angives et mønster, udføres
handlingen for hver eneste inputlinje. Hvis der kun angives et
mønster, består defaulthandlingen af __print__-udsagnet.

For at udskrive en fil kan man således nøjes med at angive en handling
med __print__-udsagnet:

```
awk '{print}' data
```

eller benytte den idiomatiske form[^1] med mønster uden handling:

[^1]: Ved 'idiomatisk' forstås den typisk meget koncise udtryksform,
  som er karakteristisk for selve sproget.

```
awk 1 data
```

I sidstnævnte, som er det korteste awk-program, man kan skrive,
evalueres betingelsen `1` og findes sand, hvorefter defaulthandlingen
__print__ eksekveres.

<!--https://stackoverflow.com/a/59473330/8860865-->

Først fortolker awk scriptet, dernæst læses filen. Awk læser ikke en
hel fil ad gangen men bid for bid -- eller rettere optegnelse
(_record_) for optegnelse. Det er variablen `RS` (_record separator_),
der definerer hvad en optegnelse er, og som udgangspunkt er den et
linjeskift (`\n`).



## Fjernelse af blanktegn i begyndelsen og slutningen af linjer

Sed-løsning:

```bash
sed 's/^[ \t]*//g;s/[ \t]*$//g' file.txt
```

Awk-løsning:

```bash
awk '{gsub(/^[ \t]+|[ \t]+$/,"")};1'
```

```bash
awk '{$1=$1};1'
```

## Reducér flere linjeskift til ét

Hvis man skal reducere flere linjeskift til ét, kan man benytte sig af
muligheden for at betragte data, ikke som standard
enkeltlinjeoptegnelser, men flerlinjeoptegnelser, ved at sætte
optegnelsesmarkøren til en tom streng ("RS=") og efterfølgende sætte
ORS til en tom linje ("\n\n"):

VIRKER IKKE!!!!

```
awk -v RS= -v ORS='\n\n' '1' file
```
Når RS-variablen sættes til en tom streng, bliver optegnelsesmarkøren
per konvention til en tom linje ("\n\n") (Arnold 2015, s. 75)

En anden måde er 

```
awk '{ /^\s*$/?b++:b=0; if (b<=1) print }' input
```
fundet på SO <https://stackoverflow.com/a/59190861/8860865>

## Slå linjer sammen mellem tags

```bash
awk '/<p [^>]+>/{f=1} f{i=i $0}/<\/p>/{{gsub(/\n/, " "); printf "%s\n", i}  i=f=""}' hjortoe-k_hans-raaskov.xml
```

## Sammenknyt linje der ender med et bestemt tegn med den næste

```
awk '/\\$/ { printf "%s", substr($0, 1, length($0)-1); next } 1' input
```

funder på SO <https://stackoverflow.com/a/30064714/8860865>

## Søg og erstat af tekststreng



Ved brug af gawk kan man søge og erstatte tekststrenge på denne måde:

```bash
awk -i inplace '{sub("oldstring", "newstring")}1' *.xml
```
Det er vigtigt at tilføje `1`, da filerne ellers vil miste al indhold
på nær det der bliver udskiftet. Således vil følgende slå fejl: 

```bash
awk -i inplace '{gsub("keystring","newstring",$0); print $0}' *.txt
```

## Indsæt før og efter match

Hvis man ønsker at indsætte linjen "new_line" _før_ matchende regexp,
kan man benytte GNU Awk på følgende måde:

```bash
awk -i inplace '/regexp/{print "new_line"}1' input.file
```
Hvis man ønsker at indsætte linjen "new_line" _efter_ matchende regexp,
kan man benytte GNU Awk på følgende måde:

```bash
awk -i inplace '/regexp/{print; print "new_line"; next}1' input.file
```

## Fjernelse af kontroltegn

Fjernelse af _form feed_

```bash
sed 's/\f//g' file.txt
```

## Print første 10 linjer af en fil

```bash
awk 'NR < 11' file.txt
```

## Udtræk YAML-blok med awk

For at kunne udtrække en YAML-blok, som begynder og slutter med `---`,
kan man køre

```bash
awk '/---/{f=!f; next} f' file
```

## Erstat blok af XML-tags

I en TEI-XML fil skulle forekomster af 

```xml
<publisher>
  <ref target="https://dsl.dk">Det Danske Sprog- og Litteraturselskab</ref>
</publisher>
```
erstattes af

```xml
<publisher xml:lang="da">Det Danske Sprog- og Litteraturselskab</publisher>
<publisher xml:lang="en">Society for Danish Language and Literature</publisher>
```

Løsning:

```awk
BEGIN {
  var="\t\t<publisher xml:lang=\"da\">Det Danske Sprog- og Litteraturselskab</publisher>\n\t\t<publisher xml:lang=\"en\">Society for Danish Language and Literature</publisher>"
}
/<publisher>/{skip=1; print var}1{if(!skip)print}/<\/publisher>/{skip=0}
```

Lavede derpå ny mappe og kopierede filerne dertil vha.

```bash
for i in *.xml; do awk -f pub.awk "$i" > changed-files/"${i%. *}"; done
```

## Print fra matchende regex til matchende regex

Selvom et rækkeviddeudtryk som `/start/,/end/` er en kort og koncis måde
at printe et interval, er det kun lidt kortere end det mere
robuste og udvidelige:

```
awk '/start/{f=1} f{print; if (/end/) f=0}'
```

For hvis man fx ønsker at udelade afgrænsningsmarkørerne, er man nødt
til at gentage sin kode og skrive:

```
awk '/start/,/end/{ if (!/start|end/) print }'
```

Ved at bruge flag som `f=1` og `f=0` kan man udelade
afgrænsningsmarkørerne således:

```
awk 'f{if (/end/) f=0; else print} /start/{f=1}' 
```



## Print fra matchende regex til enden af filen

<!--https://stackoverflow.com/questions/17908555/printing-with-sed-or-awk-a-line-following-a-matching-pattern/17914105#17914105-->

En idiomatisk løsning til at printe fra og med et match til og med
enden af filen kan ses her:

```bash
awk '/regex/,0' file.txt
```

Samme resultat kan opnås ved at skrive: `awk '/regexp/{f=1}f'
file.txt`. `f` er kort for _fundet_. Det fungerer som et boolsk flag
til '1' (sandt), når en streng, der matcher det regulære udtryk i
input.

Sidstnævnte idiom kan bruges til bortfiltrere linjer efter et
matchende regulært udtryk som items i en yaml-liste nedenfor. Har man
en liste som

```yaml
other.brokers:
    - "soddkgæsdk"
    - "kfdsjlkdlj"
    - "fldkflkglk"
kafka.brokers:
    - "node003"
    - "node004"
other.brokers:
    - "dsjkgjsklk"
    - "sdlæslskks"
kafka.brokers:
    - "node005"
other.brokers:
    - "slkslgkssæ"
```
vil dette script

```bash
awk 's{if(/\s*-\s*"[^"]*"/) next; else s=0} /kafka.brokers:/{s=1}1' file
```

give følgende resultat:

```yaml
other.brokers:
    - "soddkgæsdk"
    - "kfdsjlkdlj"
    - "fldkflkglk"
kafka.brokers:
other.brokers:
    - "dsjkgjsklk"
    - "sdlæslskks"
kafka.brokers:
other.brokers:
    - "slkslgkssæ"
```

Scriptet kan forklares på følgende måde: Hvis det regulære udtryk
matcher, spring videre til næste linje (`if(/\s*-\s*"[^"]*"/)
next`). Tjek om udtrykket matcher, men kun hvis `s` er positiv
(`s{if(/\s...`). Hvis udtrykket matcher, sættes `s` til positiv
(`/kafka.brokers:/{s=1}`). Print linjenhvis den ikke springes over
(`1`). Vvis `s` var positiv, men det regulære udtryk ikke matcher,
nulstil `s` (`s{... else s=0}`).

<!--Eksemplet fundet her: https://stackoverflow.com/questions/41214611/bash-sed-query-to-edit-yaml-file -->

## Print efter matchende regex til enden af filen

For at printe alle forekomster _efter_ et match til og med enden af
filen kan man skrive:

```bash
awk 'f;/regexp/{f=1}' file.txt
```

## Print hver n. forekomst

Skær de første 9 tegn af hver 4. linje:

```bash
awk 'FNR%4==0{print substr($0, 10);next}1' data.txt
```

## Print den n. forekomst efter matchende regex

```bash
awk 'c&&!--c;/regexp/{c=N}' file.txt
```
Hvis 'c' er ikke-nul, dekrementer den og test om den er nul. Hvis 'c'
er begyndt som et positivt tal, vil den associerede handling (at
udskrive linjen) blive udført efter at der er talt ned fra tallet til
nul. Betingelsen _hvis 'c' er ikke-nul_ sættes ind for at undgå at 'c'
fortsætter i negative tal.


## Print alle linjer på nær den n. forekomst efter matchende regex

```bash
awk 'c&&!--c{next}/regexp/{c=N}1' file
```

## Sammenføj alle linjer

```bash
awk '{printf $0;}' file
```
## Omgiv afsnit med <p>-elementer

```bash
awk -v RS="" '{printf "%s", "<p>\n" $0 "\n</p>" RT}' file.txt
```

<!--https://unix.stackexchange.com/questions/364112/sed-paragraph-tags#364121-->

## Fjern alle HTML-tags

Hvis man ikke har behov for at parse HTML, men blot ønsker at fjerne
al opmærkning fra teksten, kan det gøres således:

```bash
awk -v RS='<[^>]+>' -v ORS= '1' file.html
```
I eksemplet her sættes ORS til null, således at awk ikke outputter et
linjeskift efter hver record. '1' er kortformen for `{print}`. 

Der er således ingen grund til at bruge substitutionsfunktioner til
bortfiltrering af tekst.

<!-- Parallelt eksempel her:

awk -vRS='(```{=html}\n)?<[^>]+>\n(```)?' -vORS= '1' awktips.md

-->

<!--https://stackoverflow.com/questions/39124144/awk-remove-characters-match-with-html-tag-->
<!--https://stackoverflow.com/a/65100553/8860865 : Parallelt eksempel med bortfiltrering af blokke som indeholder et match-->
<!--https://stackoverflow.com/a/55877538/8860865 : Ed Mortons forklaring af hvordan OFS fungerer i awk-->

## Lav liste over historik og sorter den

```bash
history | awk '{a[$2]++ } END{for(i in a){print a[i] " " i}}' | sort -rn | head
```

<!--Validering af XML-filer bruger samme fremgangsmåde:

val * | awk -F: '{a[$5]++}END{for(i in a){print a[i] " " i}}' | sort -nr | head -20

-->

## Udskriv de kolonner, du har brug for

Hvis man har masser af data i regneark og kun ønsker at sammenligne
nogle af kolonnerne, kan man bruge dette nyttige idiom. Givet denne tabel:

```
foo bar baz
1   a   alpha
2   b   beta
3   c   gamma
```
Ønsker man kun at se kolonnerne __foo__ og __baz__, kan man skrive et
awk-program på denne måde:

```
awk '
NR==1 {
    for (i=1; i<=NF; i++) {
        f[$i] = i
    }
}
{ print $(f["foo"]), $(f["baz"]) }
' file
```

<!--https://unix.stackexchange.com/a/359699/450236-->

## Tæl ord

Lav en (primitiv) liste over de 10 hyppigst forekommende ord:

```bash
awk '{for(i=1;i<=NF;i++)freq[$i]++}END{for(w in freq){print w " " freq[w]}}' file | sort -rn -k2 | head
```

## Omdøb filer og tag højde for dubletter

Givet en mappe med filer med følgende navngivning:

```bash
120722-nb-haraldb.tex
120809-hannaa-nb.tex
120809-hannaa_haraldb-nb_mnb.tex
120809-haraldb-nb.tex
121111-haraldb-nb.tex
```

og vi ønsker følgende tabel med de oprindelige filnavnene og de ønskede filnavne:

```
120722-nb-haraldb.tex 120722001
120809-hannaa-nb.tex 120809001
120809-hannaa_haraldb-nb_mnb.tex 120809002
120809-haraldb-nb.tex 120809003
121111-haraldb-nb.tex 121111001
```

I så fald kan vi bruge følgende awk-program:

```bash
ls | awk -F- '{ print (($1 in a) ? $0 " " $1"00"a[$1]+1 : $0 " " $1"001") ; a[$1]++ }' > mapping-table.txt
```
Man vil herefter kunne køre:

```bash
while read -r old new; do
    cp "$old.png" "$new.png"
done < mapping-table.txt
```
<!--https://stackoverflow.com/a/65837117/8860865 -->

<!--Denne er beslægtet:

```
ls | awk -F- 'cnt[$1]++{$1=$1" variant "cnt[$1]-1} 1'
```
-->
<!-- https://stackoverflow.com/questions/29977511/how-to-rename-duplicate-lines-with-awk>-->

## Nummerering af kapitler

Med udgangspunkt i en XML-fil skal der indsættes kapitelnumre som
kapiteloverskrift. Hver gang vi støder på `<reg>` skal dette erstattes med
`<reg> Kap. 1`, `<reg> Kap. 2` osv. Dette kan klares med denne oneliner:

```awk
awk '/<reg\/>/{sub(/<reg\/>/, "<reg>Kap. "++i "<\/reg>")}1' fil.xml
```

## Arbejde med afsnit

I en tekst som denne:

```
Having said that, let's take a look at the awk equivalents for the
posted sed examples below
```
kan man matche over flere linjer ved at sætte `RS` til NULL 

```
awk -v RS= '/for the\nposted/ gsub(/for the\nposted/, "noget andet")' file
```
Som eksempel på bortfiltrering af 
Som eksempel på substitution af blokke i afsnitsordnet tekst troede
jeg kan følgende fremhæves:

```
awk -vRS= 'gsub(/(```{=html}\n)?<[^>]+>\n(```)?/, "\n")' awktips.md
```

<!--https://stackoverflow.com/questions/32000689/awk-filtering-out-paragraphs-->
<!--https://unix.stackexchange.com/a/136305-->
<!--https://www.gnu.org/software/gawk/manual/html_node/Multiple-Line.html-->
<!--https://stackoverflow.com/a/25298307/8860865-->

## Fizzbuzz

Fizzbuzz er et klassisk jobsamtalespørgsmål. Opgaven lyder typisk på
noget i retning af:

> Implementer en funktion, der 
>  - givet et tal, der er deleligt med 3, printer 'fizz'
>  - givet et tal, der er deleligt med 5, printer 'buzz'
>  - givet et tal, der er deleligt med både 3 og 5, printer 'fizzbuzz'
>  - og ellers udskriver tallet

Med awk kan opgaven løses på følgende måde:

```bash
for i in $(seq 10 16); do echo $i | awk '$0%3==0{s="fizz"}$0%5==0{s=s"buzz"}s{print s}!s{print$0}'; done
```
## Pascals trekant

```bash
awk 'BEGIN{for(i=0;i<6;i++){c=1;r=c;for(j=0;j<i;j++){c*=(i-j)/(j+1);r=r" "c};print r}}'
```

## Kartesisk produkt

Det kartesiske produkt af to mængder _A_ og _B_ er mængden af alle par
(_a_,_b_), hvor _a_ ∈ _A_ og _b_ ∈ _B_. For eksempel således
implementeret:

Givet to filer:

```bash
$ cat file1
a
b
$ cat file2
c
d
e
```

giver:

```bash
awk 'NR==FNR {a[++n]=$0; next} {for (i=1; i<=n; ++i) print $0 "," a[i]}' file2 file1
a,c
a,d
a,e
b,c
b,d
b,e
```
<https://stackoverflow.com/a/68715535/8860865>

## Sortering af flerlinjeoptegnelser

Data som disse

```
[n3]
lemma = "jek"
body = "lksdæs"

[n1]
lemma = "jek"
body = "lksdæs"

[n2]
lemma = "jek"
body = "lksdæs"

```

kan sorteres alfabetisk vha. dette program

```awk
BEGIN {
  RS=""
  ORS="\n\n"
  FS=OFS="\n"
  PROCINFO["sorted_in"]="@val_str_asc"
}

{
  a[NR] = $0
}

END {
  for(i in a) print a[i]
}
```

med dette resultat:

```
[n1]
lemma = "jek"
body = "lksdæs"

[n2]
lemma = "jek"
body = "lksdæs"

[n3]
lemma = "jek"
body = "lksdæs"
```

## Sammenlign filer

Dette SO-svar kan være et godt udgangspunkt for at lave en ordtælling
der ser bort fra stopord, jf <https://stackoverflow.com/a/32747544/8860865>

## Generer liste over en bestemt slags XML-element

Sådan genereres en liste over hvordan elementet `<persName>` er brugt
i Herman Bangs breve:

```bash
cat *.xml | tr -s ' ' '\n' | awk '/<persName/{f=1}f{i=i $0 " "}/<\/persName/{printf "%s \n", i; i=f=""}'
```
som giver

```
<persName>Peter Nansen</persName>
<persName>Herman Bang</persName>
<persName>Vilhelm Greibe (dansk vicekonsul i Hamburg)</persName>
<persName key="ferslew_chr_1838">Ferslev</persName><!--<ref
```

Ulempen her er at jeg ikke på nogen nem måde kan få glæde af awks
FILENAME-variabel, fordi awk læser fra stdin.

En anden mulighed der tillader dette, men ikke får hele elementet med, er:

```bash
awk -F'<\/?persName[^>]*>' '/<persName/{print $2, " - " FILENAME}' *.xml
```
som giver

```
Peter Nansen  - 18801220001.xml
Herman Bang  - 18810000001.xml
Vilhelm Greibe (dansk vicekonsul i  - 18810000001.xml
Ferslev  - 18810000001.xml
...
```

Denne her kan også noget:

```
awk '/<persName/{f=1}f{gsub(/^[^<]*<persName>/, " "); print FILENAME, $0; if(/<\/persName/) f=0}' *.xml | less
```
Men denne her er nok bedst:

```
awk -vRS="<persName[^>]+>" '/<TEI/{f=!f;next}f{sub(/<\/persName>/, ""); print $1, $2}' *.xml
```

Med et rent element kan man generere en smuk liste

```
awk -vRS="<persName>" '/<TEI/{f=!f;next}f{sub(/<\/persName>/, ""); print $1, $2}' *.xml
```

# Udtræk data i XML/HTML tags

```
awk -v RS='\\s*</?li>\\s*' '!(NR%2)' input.html
```
<https://stackoverflow.com/a/68511308/8860865>

Ovenstående er imidlertid problematisk ved behandling af flere filer.
Her er det bedre at benytte variablen `FNR`, da denne begynder
optælling af fortegnelser forfra for hver fil:

```
awk -vRS='\\s*</?persName[^>]*>\\s*' '!(FNR%2)' *

```

# Udtræk data fra XML/HTML

Givet data som disse

```html
<div class="result">
  <div class ="item">
    <div class="item_title">ITEM 1</div>
  </div>
  <div class="item_desc">
  ITEM DESCRIPTION 1
  </div>
</div>
<div class="result">
  <div class="item">
    <div class="item_title">ITEM 2</div>
  </div>
  <div class="item_desc">
  ITEM DESCRIPTION 2
  </div>
</div>
```
kan man bruge denne awk 

```bash
awk -F '<[^>]+>' '
    found { sub(/^[[:space:]]*/,";"); print title $0; found=0 }
    /<div class="item_title">/ { title=$2 }
    /<div class="item_desc">/  { found=1 }
' file
```
til at udtrække disse data:

```
ITEM 1;ITEM DESCRIPTION 1
ITEM 2;ITEM DESCRIPTION 2
```

<https://stackoverflow.com/a/23748851/8860865>

# Udtræk af streng efter specifik regexp

Med dette script 

```
awk '                                   ##Starting awk program from here.
BEGIN{print "title"}
match($0,/%29\/">[^/]*/){               ##Using match function to match regex %29\/"> till / here.
  print substr($0,RSTART+6,RLENGTH-5)   ##Printing sub string here.
}
'  Input_file                           ##Mentioning Input_file name here.
```

kan man med følgende input

```
<html>
<head><title>Index of /Data/Movies/Hollywood/2016_2017/</title></head>
<body bgcolor="white">
<h1>Index of /Data/Movies/Hollywood/2016_2017/</h1><hr><pre><a href="../">../</a>
<a href="1%20Buck%20%282017%29/">1 Buck (2017)/</a>                                     25-Nov-2019 10:25       -
<a href="1%20Mile%20to%20You%20%282017%29/">1 Mile to You (2017)/</a>                              25-Nov-2019 10:26       -
<a href="1%20Night%20%282016%29/">1 Night (2016)/</a>                                    25-Nov-2019 10:27       -
</pre><hr></body>
</html>
```
udtrække dette:

```
title
1 Buck (2017)/
1 Mile to You (2017)/
1 Night (2016)/
```

# Lav JSON-liste af TEI pb-element

Sådan kan man lave JSON-liste ud af en TEI-XML-fil med facs- og
n-attributter:

```awk
BEGIN { print "["}
/<pb[^>]+>/{
  sub(/rend="="[ ]+/, "")
  match($0, /n="[^"]+"/)
  print "\t{"
  print "\t\t\"name\" : "substr($0,RSTART+2,RLENGTH-2)","
  match($0, /facs="[^"]+"/)
  print "\t\t\"file\" : "substr($0,RSTART+5,RLENGTH-6)".jpg\""
  print "\t},"}
END { print "]" }
```

# Udtræk værdi fra JSON

Givet input som dette

```
"versionName": "1.1.5000-internal",
```

kan værdien uden _-internal_ udtrækkes således:

```bash
awk 'sub(/^[[:space:]]*"versionName"[[:space:]]*:[[:space:]]*/,"") {
      sub(/[[:space:]]*,[[:space:]]*$/,""); gsub(/^"|"$/,"");
      match($0,/-internal/);print substr($0,1,RSTART-1)}' input
```

# Udtræk værdier fra YAML-liste

Givet dette

```yaml
Market: open
Season: fall
Fruits:
- apple
- orange
- banana
- grapes
Vendors: 7
Buyers: 5
Vegetables:
- tomato
- carrot
- broccoli
```

giver denne awk

```bash
$ awk '$1 == "-"{ if (key == "Fruits:") print $NF; next } {key=$1}' file
apple
orange
banana
grapes
```

<https://stackoverflow.com/a/66188243/8860865>

# Udtræk tilstødende tegn

For at finde bogstavet til venstre for 'e' i denne liste

```
test
check
grep
```
kan man bruge
```
awk '/e/{print substr($0,index($0,"e")-1,1)}' Input_file
```
og få

```
t
h
r
```
jf. <https://stackoverflow.com/q/67068325/8860865>


# Beregn gennemsnit af forekomster

Givet følgende indhold

```
A   0.234
B   0.345
A   0.43
B   0.323
A   0.78
B   0.45
F   0.89
L   0.34
F   0.21
L   0.3
F   0.1
```

kan gennemsnittet beregnes således:

```
awk '{sum[$1] += $2; ++freq[$1]} END {for (i in freq) printf "%s\t%.4f\n", i, sum[i]/freq[i]}' file
```

```
A   0.4813
B   0.3727
F   0.4
L   0.32
```

<https://stackoverflow.com/a/67252700/8860865>

# Idiomatisk brug af array 'seen'

Et almindeligt idiom i awk er `!seen[$0]++`, som typisk bruges til at
fjerne dubletter. Her etableres et array kalder 'seen', her indekseret af
sidste felt (`$0`). Arrayet har, når det testes første gang, værdien
'0' ved hvert indeks, og når det testes igen med den samme streng har
det værdien '1' osv. som følge af postinkrement.

>By convention we always use `seen` (rather than x or anything else) as
>the array name when it represents a set where you want to check if
>it's index has been seen before

<https://stackoverflow.com/a/41050356/8860865>

Sådan kan man printe unikke linjer med udgangspunkt
i første felt:

```bash
awk -F, '!seen[$1]++' Input.csv
```
Fx

```bash
$ cat Input.csv
10,15-10-2014,abc
20,12-10-2014,bcd
10,09-10-2014,def
40,06-10-2014,ghi
10,15-10-2014,abc
```
```
$ awk -F, '!seen[$1]++' Input.csv
10,15-10-2014,abc
20,12-10-2014,bcd
40,06-10-2014,ghi
```
<https://stackoverflow.com/a/26867359/8860865>
<https://stackoverflow.com/a/21201501/8860865>


Sådan kan man printe unikke blokke med indeksering af sidste felt:

```bash
awk '$1=="Func"{ f=!seen[$NF]++ } f' file
```

Fx

```bash
Func (a1,b1) abc1
{
xyz1;
    {
        xy1;
    }
xy1;
}

Func (a2,b2) abc2
{
xyz2;
    {
        xy2;
        rst2;
    }

xy2;
}

Func (a1,b1) abc1
{
xyz1;
    {
        xy1;
    }
xy1;
}

```

<https://stackoverflow.com/a/49321661/8860865>

Sådan kan rydde en map givet følgende tabel:

```bash
1 2 3 1 2
5 6 7 7
19 20 20
```

```bash
$ awk '{
    delete(seen)
    for ( i=1; i<=NF; i++ ) {
        if ( !seen[$i]++ ) {
            printf "%s%s", (i>1 ? OFS : ""), $i
        }
    }
    print ""
}' file
1 2 3
5 6 7
19 20
```
<https://stackoverflow.com/a/42301597/8860865>

Tæl linjer

```bash
$ awk '{sub(/:.*/,"",$5)} !seen[$5]++{unq++} END{print unq}' file
```

Gawk

```bash
$ awk '{seen[$5]} END{print length(seen)}' file

```

<https://stackoverflow.com/a/32855180/8860865>

- [The GNU Awks User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html)
- [10 Awk Tips, Tricks and Pitfalls](https://catonmat.net/ten-awk-tips-tricks-and-pitfalls)
- [Idiomatic awk](https://backreference.org/2010/02/10/idiomatic-awk/)
- [Awk -- an introduction and tutorial by Bruce Barnett](https://www.grymoire.com/Unix/Awk.html)
- [Awk tips](https://web.archive.org/web/20160129181705/http://awk.info/?Tips)
- [Awk Repl](https://awk.js.org/)
- [Handy one-line scripts for Awk](http://www.pement.org/awk/awk1line.txt)
- [Awk Tips](http://awk.freeshell.org/AwkTips)
- [Data mining: in three languages](http://paddy3118.blogspot.com/2007/01/data-mining-in-three-language05.html)
- [Using awk for data science](http://john-hawkins.blogspot.com/2013/09/using-awk-for-data-science.html)
