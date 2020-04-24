---
title: HTML linting
theme: moon
revealOptions:
    transition: 'none'
css: slides.css
---

## HTML linting

---

### Motivation 1: Source code consistency

- We have tools to autoformat most of our code, plugged into CI or git hooks

  - Well-formatted source code is easier to work with
  - Auto-formatting tools (`ocamlformat`, `black`) save time and cognitive load

- In CI we check most of our code against all the linters we can find

- But not our Jinja2 templates

---

### Motivation 2: HTML compliance

- About a year ago I spent a few weeks working on accessibility / compliance
  for web-findings

- Would be good to be able to "lock that in" in CI

- Would be good to know our pages are correct

Note: We don't (shouldn't) care about formatting the finished HTML, indeed we
should consider minimising and/or compressing it (if we don't already).

---

## Wish list

- Jinja template checking / linting
- Jinja template autoformatting
- HTML correctness checking
- HTML accessibility checking

---

## 1. Python html-linter project

- <https://github.com/deezer/html-linter>
- Follows Google HTML standard
- Uses the `template-remover` project so you can apply it to templates
- Doesn't like the underscores in our BEM-style CSS classes, but you can turn
  that off with the `--disable=names` option

---

### Problems

1. Mostly reports indentation errors where it has removed a pair of enclosing
   template tags:

   ```html
   <p>correctly indented HTML</p>
   {% call some_macro() %}
     <p>HTML indented relative to the Jinja code...</p>
   {% end call %}
   <p>more correctly indented HTML</p>
   ```

   Here it will report that the paragraph is indented too far.

   - This is because `template-remover` just removes the surrounding lines: not
     very smart.)

   - It seems to be a bit inconsistent: seems OK with

     ```html
     {% if ... %} ... {% end if %}
     ```

     but not with

     ```html
     {% call ... %} ... {% end call %}
     ```

---

### Problems continued

2. Crashes on several of our templates.

   - Turns out that it's `template-remover` that crashes, not `html-linter`
     itself.

---

### What it finds

- Weirdly suggests removing a whole bunch of "optional" closing tags for
  paragraphs, lists etc.
- Finds some quote mark inconsistencies
- Finds some whitespace inconsistencies
- Some warnings about setting protocols not being necessary
- Suggests changing `doctype` to `DOCTYPE`

### Verdict

Not worth the effort...

---

## Aside: the problem is template-remover

```bash
ian@ian-laptop:~/web-findings$ remove_template.py app/web/templates/report-diffs.html.j2

                      0000000000000000000000000000000000000000000000000000000000000000

  site--fixed site--with-subheader

  00000000000000000000000000000000000000000000000000000000000000000000000

  0000000000000000000000000000000

  <div class="report-card">
    <a class="report-card__detail block-link" href="000000000000000000000" data-scroll-element>
      <div class="report-card__top-row">
        <div class="report-card__name searchable-list__element-term">00000000000000000</div>
      </div>

        <div class="report-card__bottom-row">
          <div class="report-card__description">
            000000000000000000000000
          </div>
        </div>

    </a>
  </div>

      <div class="report-card">
        <a class="report-card__detail block-link" href="000000000000000000000000000000">
...
```

(Lots of blank lines deleted)

---

## 2. HTML Tidy

- <http://www.html-tidy.org/>
- `sudo apt install tidy`
- Major project, been around for many years
- VSCode and Vim plugins

But... these don't work on Jinja2 templates, only on HTML

---

## 2(a) HTML Tidy via git-lint

- <https://github.com/sk-/git-lint>
- Generic linter-caller integrated with git...
- By default only operates on changed files, but can over-ride with `-f` option
- Also uses template-remover

Still nothing useful:

```bash
ian@ian-laptop:~/web-findings$ git-lint -f app/web/templates/report-diffs.html.j2

... Warnings about use of deprecated / dangerous library calls...

Linting file: app/web/templates/report-diffs.html.j2
line 23, col 3: Warning: plain text isn't allowed in <head> elements
```

Meaningless: The line number given does not relate to the source file.

Very likely it's still `template-remover` that's the problem. It's just too
dumb.

---

## 2(b) Tidy via Curl

Give up on the templates. Work with the finished pages instead...

- Need a running instance of web-findings
- Need a session cookie

Could also potentially integrate this with Selenium instead?

---

### Getting a cookie

1. While logged in, go to developer tools (Ctrl-Shift-I)
2. Go to Network tab
3. Maybe need to reload the page
4. Right click on the entry for the page itself
5. Select "Copy -> Copy as cURL"
6. Can delete everything except `-H 'Cookie: JSESSIONID...'` (long)

Should also be possible to get a cookie without the browser, by sending a `POST`
request with a valid username and password to the login page.

---

### Tidy / Curl

Once you have the cookie:

```bash
curl 'http://localhost:5000/report_diff/91/93' -H 'Cookie: ...' | tidy -eq
```

- `-e` says just report errors and warnings
- `-q` says be quiet

Normally `tidy` transforms HTML pages, cleans them up.

If running locally, about half the HTML will come from the Flask debug toolbar...

... and in the case I tested, *all* the warnings came from there.

---

### Tidy / Curl continued

TO DO:

- Try introducing some errors into templates and see what happens
- Some discussion of the kind of errors it reports

---

## 3. VSCode "Better Jinja" extension

Provides code highlighting / colouring but no formatting

(Include a screenshot)

Must be similar for Vim...?

---

## 4. Jinja-ninja

Only tool I found that generated meaningful results for templates

```bash
ian@ian-laptop:~/web-findings$ pip install jinjaninja
...
ian@ian-laptop:~/web-findings$ jinja-ninja app/web/templates/report-diffs.html.j2
app/web/templates/report-diffs.html.j2:7:86 Block closures should also have names `{% endblock %}`
app/web/templates/report-diffs.html.j2:11:0 Block closures should also have names `{% endblock %}`
app/web/templates/report-diffs.html.j2:15:0 Block closures should also have names `{% endblock %}`
app/web/templates/report-diffs.html.j2:19:0 Block closures should also have names `{% endblock %}`
app/web/templates/report-diffs.html.j2:56:0 Block closures should also have names `{% endblock %}`
```

A bit fussy...

---

### Jinja-ninja continued

TO DO: Try inserting errors into a template

- See if our existing tests catch it
- See if jinja-ninja catches it

- BUT... this only checks template correctness
- potentially worthwhile, but not really one of our goals here

---

## 5. Check_my_ninja.py

```python
import sys
from jinja2 import Environment

env = Environment()
with open(sys.argv[1]) as template:
    env.parse(template.read())
```

If the template parses, exits with success, otherwise throws
`jinja2.exceptions.TemplateSyntaxError`

- Super-simple
- Should be easy to incorporate into our test framework
- BUT... only checks template correctness
- potentially worthwhile, but not really one of our goals here

---

TO DO: Another possibility might be to try `curl` plus the Python `html-linter`

---

## Conclusions

- `template-remover` sucks big-time
- Other tools are potentially worthwhile, but without something to pass from
  template to HTML, they're mostly useless
- `cURL` + `tidy` seems worthwhile
- The VSCode "Better Ninja" extension is a small help
