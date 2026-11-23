NAME = Harisankar_Binod

$(NAME).pdf: $(NAME).tex
	latexmk -pdf -interaction=nonstopmode $(NAME).tex
	cp $(NAME).pdf ../static/$(NAME).pdf

clean:
	latexmk -C

.PHONY: clean
