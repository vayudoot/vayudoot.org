---
layout: post
title:  "Life as summarized in Lisp"
date:   2019-06-01 12:45:49 +0530
categories: others
---

You have a right to perform your prescribed duty, but you are not entitled to the fruits of action. - *[Bhagavad Gita](https://en.wikipedia.org/wiki/Bhagavad_Gita)*

```common-lisp
(define life
    (lambda (karma)
        (cond
            ((null? karma) 'moksha)
            (else (life (cdr karma))))))

(define karma '(ma phaleshu kadachana))

(life karma)
```
