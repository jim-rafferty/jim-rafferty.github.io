
This is a little bloglet about converting LaTeX documents to docx, which I don't do this often enough to remember how to do it. Many of us will have that one collaborator that will always reply back and ask for a manuscript draft in a Word format. Most of my colleagues nowadays are happy with LaTeX, but there's almost always one! 

Pandoc makes it super easy to convert a LaTeX source file to docx format, and can be persuaded to include references, most internal cross references, figures, tables etc. [This blog](https://medium.com/@zhelinchen91/how-to-convert-from-latex-to-ms-word-with-pandoc-f2045a762293) from Zhelin Chan got me most of the way there, but there are a few extra steps I needed to get everything working.

Firstly, I needed to install Pandoc. I'm using a Mac these days so homebrew made installation straightforward. You also need an extension to Pandoc called `pandoc-crossref`

```
brew install pandoc pandoc-crossref
```


With these applications installed, the command to convert from `file.tex` to `file.docx` is 

```
pandoc --filter pandoc-crossref --citeproc file.tex --bibliography=references.bib --csl=vancouver.csl -o file.docx
```


I've also pointed the application at `vacouver.csl` to format the references in the Vancouver style. 