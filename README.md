# Can-you-Quant-this-
Optiver Challenge
## Can you Quant This?
**Quantitative Puzzle involving Markov-Chains**

[Project Link- https://hammadrkh.wixsite.com/portfolio/projects/can-you-quant-this%3F)

This is a typical interview assesment question that I came across for a Graduate Quantitative Researcher position. Many of these positions put an empasis on Phd to masters level proficiency but in my belief this is a problem thats scarier than it looks but can be solved with a Markov Chain Process as we will assume that there is no memory of past events. If it was not for sake of time considering this would be in a timed interview, im sure many asssumed or employed a Monte Carlo Simulation by incorporating repeated samplings of random walks over a set of probabilities. 


A trick I have during my journey of learning/sharpening my skills is that it's important to not just assume or know how to solve the problem, but actually put emphasis on understanding what the problem is asking as well as what actually needs to be solved within it. I didnt see any answers for this online, to which even individuals with Phd's seemed to have trouble after part 1 of the question, so I figured maybe a diverse  outlook/approach from an outsider would do the trick!


Can you solve this puzzle? 

An ant leaves its anthill in order to forage for food. It moves with the speed of **10cm per second**, but it doesn't know where to go, therefore every second it moves randomly 10cm directly north, south, east or west with equal probability.

If the food is located on east-west lines **20cm** to the north and 20cm to the south, as well as on north-south lines **20cm** to the east and 20cm to the west from the anthill, how long will it take the ant to reach it on average?

What is the average time the ant will reach food if it is located only on a diagonal line passing through **(10cm, 0cm)** and **(0cm, 10cm)** points?

Can you write a program that comes up with an estimate of average time to find food for any closed boundary around the anthill? What would be the answer if food is located outside an area defined by **( (x – 2.5cm) / 30cm )2 + ( (y – 2.5cm) / 40cm )2 < 1** in coordinate system where the anthill is located at **(x = 0cm, y = 0cm)?** Provide us with a solution rounded to the nearest integer.


# The goal of this notebook

Here are my solutions for the Optiver challenge regarding the amount of time it would take an ant to encounter food in various scenarios. Specifically, the question states:

An ant leaves its anthill in order to forage for food. It moves with the speed of **10 cm per second**, but it doesn't know where to go, therefore every second it moves randomly **10 cm** directly north, south, east or west with equal probability.

First lets take a deep breath and import the libraries we need!

```Python
import numpy as np
from abc import ABC, abstractmethod
from progressbar import ProgressBar
```

To solve the various subquestions, we first define a **base_ant_random_walk class**, which we will use as a foundation for the various wandering ant scenarios (such as encoding the wandering behavior).


```Python
class base_ant_random_walk(ABC):
    def __init__(self, ant_number=100000, start_x=0.0, start_y=0.0, dt=1, v0=10):
        self.ant_number = ant_number                           # Number of ants we release
        self.start_x, self.start_y = start_x, start_y          # starting x and y coordinate
        self.xy_positions = self.initialize_position_array()   # initialized array of particle positions
        self.time = self.initialize_time_array()               # initialized array of particle time
        self.dt = dt                                           # integration timestep (seconds)
        self.v0 = v0                                           # walk speek (cm / s)
        
    def initialize_position_array(self):
        # Initializing the array in which we store the ants' x and y positions
        xy_positions = np.zeros(shape=(self.ant_number, 2))
        xy_positions[:, 0] = self.start_x
        xy_positions[:, 1] = self.start_y
        
        return xy_positions
    
    def initialize_time_array(self):
        return np.zeros(shape=(self.ant_number))
        
    def random_walk(self, xy_positions):
        # Generate an array with random integers between 0 - 3 which will set the direction of the random walks
        walk_direction = np.random.randint(low=0, high=4, size=self.ant_number)
        
        # If walk_direction == 0, move north by v0 * dt
        north = np.where(walk_direction == 0)[0]
        xy_positions[north, 1] += self.dt * self.v0
        # if walk_direction == 1, move south by v0 * dt
        south = np.where(walk_direction == 1)[0]
        xy_positions[south, 1] -= self.dt * self.v0
        # if walk_direction == 2, move east by v0 * dt
        east = np.where(walk_direction == 2)[0]
        xy_positions[east, 0] += self.dt * self.v0
        # if walk_direction == 3, move west by v0 * dt
        west = np.where(walk_direction == 3)[0]
        xy_positions[west, 0] -= self.dt * self.v0
        
        return xy_positions
    
    def calculate_walk(self, steps):
        for i in ProgressBar()(range(steps)):
            # Calculate the random walk procedure
            self.xy_positions = self.random_walk(self.xy_positions)
            # Update the time tracker of each ant
            self.time[~np.isnan(self.xy_positions[:, 0])] += 1
            # Set to np.nan all particles that are at or have crossed the boundary condition
            at_boundary = self.boundary_condition()
            self.xy_positions[at_boundary, :] = np.nan
            
        return self.xy_positions, self.time

    @abstractmethod
    def boundary_condition(self):
        # Define the specific condition when the ant encounters food
        pass
    
    def mean_travel_time(self, steps=1000):
        self.xy_positions, self.time = self.calculate_walk(steps=steps)
        # Determine the particles that are at the food
        at_food = np.isnan(self.xy_positions[:, 0])
        # Calculate the mean time
        mean_time = self.time[at_food].mean()
        std_time = self.time[at_food].std()
        error_time = std_time / np.sqrt(np.sum(at_food))
        
        # Calculate the number of ants that have reached the food
        at_food_percentage = np.sum(at_food) / self.ant_number * 100
        
        str_format = steps, mean_time, error_time
        print('After {} seconds, it takes the ant {:.2f}±{:.2f} seconds to encounter food.'.format(*str_format))
        print('In this time, {:.2f}% of ants have reached the food.\n'.format(at_food_percentage))  
```
## Question 1
**The first question states:**

If the food is located on east-west lines **20 cm** to the north and 20 cm to the south, as well as on north-south lines **20 cm** to the east and 20 cm to the west from the anthill, how long will it take the ant to reach it on average?
We have already prescribed the random walk behavior in base_ant_random_walk, so to solve this question numerically we just need to specify the boundary condition. **If |x| ≥ 20 or |y| ≥ 20**, the ant will have reached or crossed the boundaries, and therefore reached the food.

```Python
class ant_random_walk_question_1(base_ant_random_walk):
    def __init__(self):
        super().__init__()
        
    def boundary_condition(self):
        # if either the absolute x or y position is >= 20, then the ant has reached food
        boundary = (np.abs(self.xy_positions[:, 0]) >= 20) | (np.abs(self.xy_positions[:, 1]) >= 20)
        return boundary
        
ant_random_walk_question_1().mean_travel_time()
 
Output():


100% (1000 of 1000) |####################| Elapsed Time: 0:00:02 Time: 0:00:02

After 1000 seconds, it takes the ant 4.50±0.01 seconds to encounter food. In this time, 100.00% of ants have reached the food.
```
 



# Question 2
**The second question states:**

What is the average time the ant will reach food if it is located only on a diagonal line passing through **(10cm, 0cm)** and **(0cm, 10cm)** points?
The boundary condition for this question is that the food is found at **x + y = 10**, which we can easily encode in the boundary_condition function.

```Python
class ant_random_walk_question_2(base_ant_random_walk):
    def __init__(self):
        super().__init__()
        
    def boundary_condition(self):
        boundary = np.nansum(self.xy_positions, axis=1) == 10
        return boundary
        
ant_random_walk_question_2().mean_travel_time(steps=100)
ant_random_walk_question_2().mean_travel_time(steps=1000)
ant_random_walk_question_2().mean_travel_time(steps=10000)
 
Output():

100% (100 of 100) |######################| Elapsed Time: 0:00:00 Time: 0:00:00 

3% (38 of 1000)     |                                            | Elapsed Time: 0:00:00 ETA: 0:00:05

After 100 seconds, it takes the ant 7.69±0.05 seconds to encounter food. 

In this time, 92.14% of ants have reached the food.


100% (1000 of 1000) |####################| Elapsed Time: 0:00:03 Time: 0:00:03 

    0% (22 of 10000)   |                                         | Elapsed Time: 0:00:00 ETA: 0:00:46d

After 1000 seconds, it takes the ant 24.89±0.29 seconds to encounter food. 

In this time, 97.43% of ants have reached the food.


100% (10000 of 10000) |##################| Elapsed Time: 0:00:36 Time: 0:00:36

After 10000 seconds, it takes the ant 76.34±1.57 seconds to encounter food. 

In this time, 99.23% of ants have reached the food.
```
 

# Question 3
**The final question comes in two parts, where the first asks:**

Can you write a program that comes up with an estimate of average time to find food for any closed boundary around the anthill?
In the current form of the code this is already achieved, as in principle any closed boundary can be specified around the anthill through the boundary_condition function in **base_ant_random_walk**. Therefore, we shall now proceed to the second part, which asks:

What would be the answer if food is located outside an defined by **( (x – 2.5cm) / 30cm )2 + ( (y – 2.5cm) / 40cm )2 < 1** in coordinate system where the anthill is located at **(x = 0cm, y = 0cm)?** Provide us with a solution rounded to the nearest integer.
The notation is not entirely clear, but we will assume that the boundary indicates a elipse centered at **(0, 0)** following the following equation: 


**((x - 2.5)/30)^2 + ((x - 2.5)/40)^2 < 1**

```Python
class ant_random_walk_question_3(base_ant_random_walk):
    def __init__(self):
        super().__init__()
        
    def boundary_condition(self):
        boundary = np.square((self.xy_positions[:, 0] - 2.5) / 30) + np.square((self.xy_positions[:, 1] - 2.5) / 40) >= 1
        return boundary
        
ant_random_walk_question_3().mean_travel_time()
 
Output():

100% (1000 of 1000) |####################| Elapsed Time: 0:00:02 Time: 0:00:02

After 1000 seconds, it takes the ant 13.99±0.03 seconds to encounter food. 

In this time, 100.00% of ants have reached the food.
```
 


**As shown above, it takes an ant around 14 seconds to reach food within this closed boundary, and we can easily calculate this for any other closed boundary. As was shown for question 2, the code also works for an open boundary, but then instead of giving the mean time for the ant to reach the food, it can give the probability the ant will reach food within a given time period.**



# Methodology
The key to this is practice, tenacity, and refusal to be intimidated. Whether you have a Phd, Masters, Bachelors, etc., education does not always equal learning and in reality considering that everyone gets asked the same question, then it is safe to assume that prestige goes out the window and being able to find the answer is the only outcome that truly matters or can be quantified at the end of the day. Stay motivated and always believe that if a problem can be solved by someone else, we all breath in the same oxygen at the end of the day so it's fair game. You can achieve whatever you see other people achieve no matter what anyone tells you!
