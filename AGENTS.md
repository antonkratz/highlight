# Communication style {{{

Your answer should be parsimonious: say everything what is necessary, but never more.

Do not hedge your statements with phrases such as "something like" or "I would try" or "it should work"; instead, if you are not confident about something, state so clearly!

Speak English. Speak normally. Do not use meaningless words if normal English words are more clear.
Typical example of undesired output: "scripts/ge/analysis_env.sh is a runtime bootstrap script for every stage wrapper". Much better alternative: "scripts/ge/analysis_env.sh is a small setup script that each stage script loads before running Python".

Do not use the verb "to live" when you mean "to be located in/at". For example, instead of "The script lives in analysis/sars-cov2-presentation-time-course/" just say "The script is located in analysis/sars-cov2-presentation-time-course/".

Do not use the verb "patch" when the verb "edit" is sufficient.

Use plain, direct English. Do not use startup jargon, engineering slang, marketing language, or vague phrases.

Avoid phrases like:
- "tightening the notebook"
- "smoke test"
- "hardening"
- "wire up"
- "bootstrap"
- "defensive"
- "operationalize"
- "leverage"
- "workflow"
- "end-to-end" unless it is technically necessary

Prefer simple alternatives:
- "make the notebook simpler"
- "small test run"
- "make it more reliable"
- "connect"
- "set up"
- "avoid failures"
- "use"
- "process"
- "from training to submission"

When describing changes:
- Say exactly what changed.
- Say why it changed.
- Say what risk or limitation remains.
- Do not use clever wording, euphemisms, or filler.
}}}

# Coding advice {{{
 - Prefer plotnine over matplotlib/seaborn. Only use matplotlib/seaborn if existing code already uses it.
 - Do not use theme_minimal() for ggplot2/plotnine, just use the standard theme.
 - Figure output files should be in PDF format.
}}}

# How to make git commits {{{

Git commits should only contains code and documentation, never raw or processed data.

Git commit messages should include one line, then an empty line, and then a conceptual decsription of what has been done, which can be as long as necessary, but not longer than necessary.
}}}

