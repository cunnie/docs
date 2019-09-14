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

Vectors and Matrices

```
v = [1 2 3] % row vector, aka 1 x 3 matrix
v = [1; 2; 3] % column vector, aka 3 x 1 matrix
v = 1:6	  % creates a vector, from 1 to 6 inclusive, with 1 increments
v =

   1   2   3   4   5   6
v = 1:0.1:2  % creates a vector, from 1 to 2 inclusive, with 0.1 increments
v =

    1.0000    1.1000    1.2000    1.3000    1.4000    1.5000    1.6000    1.7000    1.8000    1.9000    2.0000

size(v) % size of v as a matrix
ans =

    1   11
length(v) % length (rows x columns) as a scalar
ans =  11
help length % displays the help about length if you're not sure
A=[1 2;3 4;5 6]
A =

   1   2
   3   4
   5   6
ones(2,3) % 2 x 3 matrix of ones
ans =

   1   1   1
   1   1   1
2*ones(2,3) % matrix of twos
zeros(2,3)  % matrix of zeros
rand(1,3) % matrix of random numbers between 0.0 and 1.0
randn(1,3)  % Gaussian distribution mean 0 std dev (variance) 1
eye(3)	% "eye" == "I" as in "identity"; 3x3, all 0s except for diagonal which is 1s
```

Histograms
```
v = randn(1,1000);
hist(v)	% histogram, default of 10 bins
hist(v,50) % 50 bins
 % let's try rolling 1 x 6 sided die 10,000 times
hist(floor(rand(1,10000)*6+1),6)
 % let's try 3 x 6 sided die
i=10000
hist(six_sided_die(i)+six_sided_die(i)+six_sided_die(i), 15);
```

Graphs
```
t = 0.0:0.01:0.98;
y1 = sin(2*pi*4*t);
plot(t,y1);
hold on; % next plot don't overwrite this plot
y2 = cos(2*pi*4*t);
plot(t,y2)
xlabel('time'); % set x-axis label
ylabel('value'); % set y-axis label
title('sin & cos vs. time'); % Set the chart title
cd '~/Downloads'; % change the current directory
print -dpng 'sin_cos.png'
 % `GenericResourceDir value does not end with directory separator`
 % means follow this link to download a patched version https://github.com/octave-app/octave-app/issues/33
```

Functions
```
function throws = six_sided_die(num_throws)
  if (nargin != 1)
    usage ("six_sided_die (num_throws)");
  endif
  throws = floor(rand(1,num_throws)*6+1);
endfunction
function die(faces, num_die, num_throws)
  if (nargin != 3)
    usage ("die (faces, num_die, num_throws)");
  endif
  throws = sum(floor(rand(num_die,num_throws)*faces+1));
   % hist()'s 2nd argument can be an array
   % 3 x 6-sided die, that would work out to 3:(3*6) = 3:18 = 3 4 5 ... 17 18
  hist(throws, num_die:(num_die*faces));
endfunction
```
