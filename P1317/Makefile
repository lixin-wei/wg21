P1317R2.pdf: P1317R2.tex
	mkdir -p generated
	pdflatex -interaction=nonstopmode -halt-on-error -file-line-error P1317R2.tex
	mv P1317R2.pdf generated/P1317R2.pdf

.PHONY: clean
clean:
	rm -f P1317R2.aux P1317R2.dvi P1317R2.log P1317R2.out P1317R2.pdf generated/P1317R2.pdf

.PHONY: all
all: P1317R2.pdf
