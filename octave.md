### Octave

Arithmetic

```octave
5+6  % '%' is a comment
ans = 11
2-3; % semicolon suppresses output
2^10;  % exponentation
```

Logical

```
1 == 2	% equals, false (0)
ans = 0
1 ~= 2	% not equals, true (1)
ans = 1
1 && 0;	% AND
1 || 0;	% OR
xor(1,0);
```

Æsthetics

```
PS1('>> ')  % set the prompt to '>> '
pi; % pi's a special variable, 𝛑, but you can set it.
ans =  3.1416
disp(pi); % print out pi, semicolon notwithstanding
 3.1416
disp(sprintf("Long 𝛑: %.9f\n",pi))
format long % moar decimals
pi
ans =  3.141592653589793
format short % fewer decimals
pi
ans =  3.1416
```

Assignment

```
a = 3; % number
b = 'hi'; % string
c = (3 >= 1); % boolean
p =  3.1416
```
