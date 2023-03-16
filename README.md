import re

string = "input/abc123def_CAIS_456ghi.v3.feedback.json.bz2"

# define the pattern
pattern = r"^input/[^\s/]+_CAIS_[^\s/]+\.v3\.feedback\.json\.bz2$"

# check if the string matches the pattern
if re.match(pattern, string):
    print("The string matches the pattern!")
else:
    print("The string does not match the pattern.")
