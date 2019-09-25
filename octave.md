### Octave

##### Arithmetic

```octave
5+6  % '%' is a comment
ans = 11
2-3; % semicolon suppresses output
2-3, 4-7, 8-10 % comma lets you put several functions on a line but doesn't suppress output
2^10;  % exponentation
```

##### Logical

```octave
1 == 2	% equals, false (0)
ans = 0
1 ~= 2	% not equals, true (1)
ans = 1
1 && 0;	% AND
1 || 0;	% OR
xor(1,0);
```

##### Matrix

```octave
[1 2 3] .^ [1 3 2] == [1 8 9] % element-by-element power operator
```

Æsthetics

```octave
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

##### Assignment

```octave
a = 3; % number
b = 'hi'; % string
c = (3 >= 1); % boolean
p =  3.1416
```

##### Vectors and Matrices

```octave
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
transpose(A) == A' % x' is a synonym for transpose(x)
ans =

  1  1  1
  1  1  1
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

##### Histograms

```octave
v = randn(1,1000);
hist(v)	% histogram, default of 10 bins
hist(v,50) % 50 bins
 % let's try rolling 1 x 6 sided die 10,000 times
hist(floor(rand(1,10000)*6+1),6)
 % let's try 3 x 6 sided die
i=10000
hist(six_sided_die(i)+six_sided_die(i)+six_sided_die(i), 15);
```

##### Graphs

```octave
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
close; % gets rid of old graph
figure(1); plot(t,y1);
figure(2); plot(t,y2); % Yay! Now I can have two plots up
subplot(1,2,1); % Divides plot 1 x 2 grid, access 1st element (left-hand side)
plot(t,y1)
subplot(1,2,2); % right-hand side
plot(t,y2)
clf; % wipes figures clean, but does not close window, blank 1 x 2 grid
close all;
A = magic(5); % magic square, sums equal across columns, rows & diags
imagesc(A); % color codes the volues: red=high blue=low
colorbar; % puts up a colorbar so you can see the values corresponding to the color
colormap gray; % grayscale not colors
colormap viridis; % default colors
colormap ocean; % current fav
```

##### Control Statements: `for`, `while`, `if`

```octave
for i=1:10,
  v(i) = 2^i;
end
indices=1:10
for i=indices, v(i) = 2^i; end
i=1; while i <= 5, v(i) = 100; i = i+1; end
i=1; while true, v(i) = 999; i = i+1;
  if i == 6,
    break; % yes, `break` works as expected
  end;
end;
if i == 6,
  disp('You are number 6');
elseif i == 2,
  disp('I am number 2');
else
  disp('Who is number 1?');
end;
```

##### Functions

Put your functions in a file named `xxx.m`. Note the `.m` suffix.

```octave
cd `~/Downloads'
function throws = six_sided_die(num_throws)
  if (nargin != 1)
    usage ("six_sided_die (num_throws)");
  endif
  throws = floor(rand(1,num_throws)*6+1);
endfunction
 % put the following in a file named `dice.m`; must match fn name
function dice(faces, num_die, num_throws)
  if (nargin != 3)
    usage ("die (faces, num_die, num_throws)");
  endif
  throws = sum(floor(rand(num_die,num_throws)*faces+1));
   % hist()'s 2nd argument can be an array
   % 3 x 6-sided die, that would work out to 3:(3*6) = 3:18 = 3 4 5 ... 17 18
  hist(throws, num_die:(num_die*faces));
endfunction
addpath('~/bin') % will look in bin for functions
 % It can return multilple values
function [x,y] = longAndLat(person) ....
```
