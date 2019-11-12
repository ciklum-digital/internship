# Homework

## Github projects

- All repositories should be `separate` and created at `your personal GitHub account`
- Name convention for the repos, next:
 
```js
// test task
const test_task = {
  name: 'js-band-test-task',
  description: 'Ciklum University: JS Band Internship. Test task'
}

// homework task 
const homework_task = {
  name: 'js-band-hw-task-1',
  description: 'Ciklum University: JS Band Internship. Homework task'
}

// final_task
const final_task = {
  name: 'js-band-final-task',
  description: 'Ciklum University: JS Band Internship. Final task'
}
```


## Homework 1

We have a new potential client, it’s one of the biggest logistics companies **“JS Band Delivery”**.
It uses two types of transporting **by trucks** and **by railway**. The company has a plan to diversify transportation types and introduce new ones, such as **Aero**.
That is why the system should be flexible to such type of changes.
The client asked about Discovery phase on this stage. The result of this phase should be a POC of the tracking system that helps them to collect data about their own transport.

What type of information they want to collect:

**Ships/Argosy:**
- id 
- Type (equal 1)
- Serial number/Name
- Produced year
- Capacity (in kg)
- Average speed (in nm)
- Count of team


**Trucks:**
- id
- Type (equal 2)
- License plate
- Produced year
- Capacity (in kg)
- Average speed (in km)
- Type of gas


**Cost of delivery:**
- Transport Type (1 or 2)
- Cost (by 1 kg of cargo)
- Cost (by 1 km of distance)


### Objectives

- Prepare the structure of the transports based on inheritance and classes;
- Prepare classes for cost of delivery;
- Prepare simple functionality for adding data to catalogs;
- Implement two methods (one inherited, one overloaded) for each type of transport:
    - overloaded `showAverageSpeed()` - should show for Track kilometers ex. 100km, for ships nautical mile for ex. 100nm
    - inherited `showCapacityInPounds()`
- Split all functionality into modules.

### Arch notes

- Nice UI isn’t required (just inputs and simple lists)
- Bundlers isn’t required you can use native ES6 modules
- Example of default data:

```js
const DEFAULT_SHIP = {
  id: 'xxxxx-qqqqq-aaaaa',
  type: '1',
  name: 'JS Band Ship',
  producedYear: '2019',
  capacity: '200000',
  averageSpeed: '20',
  countOfTeam: '83'
};

const DEFAULT_TRUCK = {
  id: 'smrta-asdad-deead',
  type: '2',
  licensePlate: 'AA1232OO',
  producedYear: '2018',
  capacity: '12000',
  averageSpeed: '110',
  typeOfGas: 'Diesel'
};
```

### Acceptance criteria
- User can fill catalogs:
    - add new ship (first form)
    - add new truck (second form)
    - add cost of delivery information (third form)
- User can see list of added transport;
- User can see list of added cost of delivery;
- Information should be stored into `local storage`;
- Functionality should be split into `modules`;
- Functionality should be implemented via `Class` statement;
- Two methods should be implemented.
