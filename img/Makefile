%.png: %.tex
	pdflatex --shell-escape $<

.PHONY: all clean
all: $(patsubst %.tex, %.png, $(wildcard *.tex))
clean:
	rm -f *.png
