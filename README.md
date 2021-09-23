# Theorem-based-LaTeX-to-ANKI
## The goal
We would to create and edit an Anki deck collectively. Furthermore, we would like to write our Deck in LaTeX and our favorit text editor. 
We would also like to write our files in the usual theorem based style and share them using a version control system such as git or the collaborative cloudbased LaTex editor overleaf.

## Current situation
There exists a LaTeX Note Importer for Anki https://tentativeconvert.github.io/LaTeX-Note-Importer-for-Anki/.
However, the importer requires its own specific syntax:

<table>
<tr>
<th>
What we want
</th>
<th>
What the LaTeX Note Importer for Anki can process
</th>
</tr>

<tr>
<td>
<pre>
\documentclass{article}
\newtheorem{definition}{Def.}
\begin{document}
\begin{definition}[Lie algebra]
  Let $G$ be a Lie group. We define the Lie algebra
  written as $\mathfrak{g}$, as the tangent space to $G$ 
  at the identity element $e$:
  \begin{equation*}
    \mathfrak{g}=T_e G
  \end{equation*}

  Examples:
  $\mathfrak{gl}(m) \equiv Mat(m)$.
\end{definition}
\end{document}
</pre>
</td>

<td>
<pre>
\documentclass{article}
\newtheorem{definition}{Def.}
\begin{document}
\begin{note}
  \xfield{Lie algebra}
  \begin{field}
  Let $G$ be a Lie group. We define the Lie algebra
  written as $\mathfrak{g}$, as the tangent space to $G$ 
  at the identity element $e$:
  \begin{equation*}
    \mathfrak{g}=T_e G
  \end{equation*}
  Examples:
  $\mathfrak{gl}(m) \equiv Mat(m)$.
  \end{field}
\end{note}
\end{document}
</pre>
</td>

</tr>
</table>

## Solutions
An initial attempt to achieve this transformation might be using the stream editor sed with regular expressions:
```
sed -e 's/\\begin{definition}\[\(.*\)\]/\\begin{note}\\xfield{Def: \1}\\begin{field}/g;s/end{definition}/end{field} \\end{note}/g' $1 > tmp_$1
```
However, one might run into the problem that the future front side of the card, momentarily being contained within the square brackets, could stretch over multiple lines. Being a unix tool, sed does one thing very well: Editing a stream line-wise. Making sed do something else is going to be hard and prone to be buggy. So instead of trying to make sed do something it is not meant to, we choose a different tool and turn to a perl-based solution:
```
function latextoanki {
perl -0777 -w -p -e 's/\\begin\{definition\}\[([^]]*)\]/\\begin{note}\n\\xfield{Def: $1}\n\\begin{field}/g;s/end\{definition\}/end\{field\} \n\\end{note}/g' main.tex > tmp_main.tex
}
```
The options for perl mean
```
-0777 pass the entire files, not merely linewise.
-w print various warnings
-p assume a loop around the program, which is executed only once in our case, but still required for evaluating the '$1' term.
-e execute code on the command line
```

We can generalize the above result for multople theorem types, such as propositions and lemmas, as follows:
```
function latextoanki {
name=(
    definition
    theorem
    proposition
    lemma
    )
diyplay_name=(
    Def.
    Theorem
    Prop.
    Lemma
    )
cp $1 tmp_$1
for j in {1.."${#name[@]}"}
do
perl -0777 -w -i -p -e "s/\\\\begin\\{${name[$j]}\\}\\[([^]]*)\\]/\\\\begin{note}\\n\\\\xfield{${diyplay_name[$j]}: \$1}\\n\\\\begin{field}/g;s/end\\{${name[$j]}\\}/end\\{field\\} \\n\\\\end{note}/g" tmp_$1
done
}
```
Now we are able to convert our files with the command `latextoanki file.tex`. The outputed import file, `imp_file.tex`, can then be importet into anki using LaTeX Note Importer for Anki.

# TL;DR
## UUID -- Preventing conflicts during updates
If you want to omit dublicating cards when changing the name field, you should further consider introducting a Universally Unique Identifier UUID.
The following code adds a UUID to the original LaTeX file, whenever you create an import file for Anki. 
```
function latextoanki {
name=(
    const
    definition
    theorem
    proposition
    lemma
    )
diyplay_name=(
	Cnst.
    Def.
    Theorem
    Prop.
    Lemma
    )
# Create backup, just in case
cp $1 bckup_$1
# UUID
for j in {1.."${#name[@]}"}
do
# Prepear adding of Universally Unique Identifiers where nonexistent
perl -0777 -w -p -i -e "s/(\\\\begin\\{${name[$j]}\\}\\[[^]]*\\])\\s*\\n\\s*(?\!\\\\uuid)/\$1\\n\\\\uuid{}\\n/g" $1
# Add a combination of unix time and line number as Unique Identifier
perl -w -p -i -e "s/\\\\uuid\\{(?=})/\\\\uuid\\{$(date +%s).$./g" $1
done
# Create the import file
cp $1 imp_$1
# Transform the import file to make it compatible with LaTeX Note Importer for Anki
for j in {1.."${#name[@]}"}
do
perl -0777 -w -i -p -e "s/\\\\begin\\{${name[$j]}\\}\\[([^]]*)\\]\\s*\\n\\s*\\\\uuid(\\{[^}]*\\})/\\\\begin{note}\\n\\\\xplain\$2\\n\\\\xfield{${diyplay_name[$j]}: \$1}\\n\\\\begin{field}/g;s/end\\{${name[$j]}\\}/end\\{field\\} \\n\\\\end{note}/g" imp_$1
done
}
```
This will require you to add a note type that considers the UUID without placing it on the front of your card. For this, go to `tool` -> `Manage note types`. Add your note type and then manage its fields by clicking on the register `Fields...` and subsequently its displaysettings by clicking on the register `Cards...`. Make sure that the field for the UUID is the first field of the card, for the ankis updater identifies cards when the first field matches. Note further, that you need to include the LaTeX packages you used in your file into your note type. This can be achieved by clicking on the register `Options...`.

Additionaly, you need to add `\newcommand{\uuid}[1]{}` to LaTeX preamble, otherwise you will get compiling errors.

If your having problems with non UTF8 characters during import, use the command `grep -axv '.*' file.tex` to track down the non UTF8 characters in your file.

## Workflow
You might want to define the function in your bashrc `v ~/.bashrc` or zshrc `v ~/.zshrc` if you intended to use it on a regular basis.


You might also be interested in installing the addons "Edit LaTeX build process". This add-on allows you to edit the command-line arguments passed to LaTeX and dvipng when creating LaTeX cards. It is not necessary to use LaTeX with Anki in most cases; it is only needed if you are trying to change the commands Anki uses to generate LaTeX output.


## TODO
### Auto Importer
This could probably be achieved by modifying
https://github.com/davidshepherd7/ankicli/blob/master/bin/anki-cli

### Figures
Is it possible to add a sepearte field to the card that is read as html `<img src="Image_00022.jpg"/>`?
See:
https://apps.ankiweb.net/docs/manual20.html#html

Is a latex internal approach prefearable?
https://tex.stackexchange.com/questions/334202/how-to-import-images-into-anki-with-latex
