# Trackinvest-assignment
 
### Adding the Routing Logic 
To keep the routing logic simple, you will route all HTTP verbs through the existing route path (with the optional id parameter). Open the services/router.js file and replace the current routing logic (lines 5-6) with the following code:
```
router.route('/employees/:id?')
  .get(employees.get)
  .post(employees.post)
  .put(employees.put)
  .delete(employees.delete);
```
### Handling POST Requests
HTTP POST requests are used to create new resources (employee records, in this case). The basic idea is to pull data out of the HTTP request body and use it to create a new row in the database. Open the controllers/employees.js file and append the code that follows.
```
function getEmployeeFromRec(req) {
  const employee = {
    first_name: req.body.first_name,
    last_name: req.body.last_name,
    email: req.body.email,
    phone_number: req.body.phone_number,
    hire_date: req.body.hire_date,
    job_id: req.body.job_id,
    salary: req.body.salary,
    commission_pct: req.body.commission_pct,
    manager_id: req.body.manager_id,
    department_id: req.body.department_id
  };

  return employee;
}

async function post(req, res, next) {
  try {
    let employee = getEmployeeFromRec(req);

    employee = await employees.create(employee);

    res.status(201).json(employee);
  } catch (err) {
    next(err);
  }
}

module.exports.post = post;
```
The getEmployeeFromRec function accepts a request object and returns an object with the properties needed to create an employee. The function was declared outside of the post function so that it can be used later for PUT requests, as well.

The post function uses getEmployeeFromRec to initialize a variable that is then passed to the create method of the employees database API. After the create operation, a "201 Created" status code, along with the employee JSON (including the new employee id value), is then sent to the client.

Now you can turn your attention to the create logic in the database API. Open the database/employee.js file and append the following code to the bottom.

```
const createSql =
 `insert into employees (
    first_name,
    last_name,
    email,
    phone_number,
    hire_date,
    job_id,
    salary,
    commission_pct,
    manager_id,
    department_id
  ) values (
    :first_name,
    :last_name,
    :email,
    :phone_number,
    :hire_date,
    :job_id,
    :salary,
    :commission_pct,
    :manager_id,
    :department_id
  ) returning employee_id
  into :employee_id`;

async function create(emp) {
  const employee = Object.assign({}, emp);

  employee.employee_id = {
    dir: oracledb.BIND_OUT,
    type: oracledb.NUMBER
  }

  const result = await database.simpleExecute(createSql, employee);

  employee.employee_id = result.outBinds.employee_id[0];

  return employee;
}

module.exports.create = create;
```
The logic above starts by declaring a constant named createSql to hold an insert statement. Note that it uses bind variables, not string concatenation, to reference the values to be inserted. It's worth repeating how important bind variables are for security and performance reasons — avoid string concatenation whenever possible.

Within the create function, an employee constant is defined and initialized to a copy of the emp parameter using Object.assign. This prevents direct modification of the object passed in from the controller. It's a shallow copy, but that's sufficient for this use case.

Next, an employee_id property is added to the employee object (configured as an "out bind") so that it contains all of the bind variables required to execute the SQL statement. The simpleExecute function is then used to execute the insert statement and the outBinds property of the result is used to overwrite the employee.employee_id property before the object is returned.

Because the oracledb module is referenced, you'll need to require that in. Add the following line to the top of the file.
```
const oracledb = require('oracledb')
```
### Handling PUT Requests
PUT requests will be used to make updates to existing resources. It's important to note that PUT is used to replace the entire resource — it doesn't do partial updates (I will show you how to do this with PATCH in the future). Return to the controllers/employees.js file and add the following code to the bottom.
```
async function put(req, res, next) {
  try {
    let employee = getEmployeeFromRec(req);

    employee.employee_id = parseInt(req.params.id, 10);

    employee = await employees.update(employee);

    if (employee !== null) {
      res.status(200).json(employee);
    } else {
      res.status(404).end();
    }
  } catch (err) {
    next(err);
  }
}

module.exports.put = put;
```
The put function uses getEmployeeFromRec to initialize an object named employee and then adds an employee_id property from the id parameter in the URL. The employee object is then passed to the database API's update function and appropriate response is sent to the client based on the result.

To add the update logic to the database API, append the following code to the db_apis/employees.js file.
```
const updateSql =
 `update employees
  set first_name = :first_name,
    last_name = :last_name,
    email = :email,
    phone_number = :phone_number,
    hire_date = :hire_date,
    job_id = :job_id,
    salary = :salary,
    commission_pct = :commission_pct,
    manager_id = :manager_id,
    department_id = :department_id
  where employee_id = :employee_id`;

async function update(emp) {
  const employee = Object.assign({}, emp);
  const result = await database.simpleExecute(updateSql, employee);

  if (result.rowsAffected === 1) {
    return employee;
  } else {
    return null;
  }
}

module.exports.update = update;
```
The update logic is very similar to the create logic. A variable to hold a SQL statement is declared and then simpleExecute is used to execute the statement with the dynamic values passed in (after copying them to another object). The result.rowsAffected is used to determine if the update was successful and return the correct value.

### Handling DELETE Requests
The last method you need to implement is DELETE which, unsurprisingly, will delete resources from the database. Add the following code to the end of the controllers/employees.js file.
```
async function del(req, res, next) {
  try {
    const id = parseInt(req.params.id, 10);

    const success = await employees.delete(id);

    if (success) {
      res.status(204).end();
    } else {
      res.status(404).end();
    }
  } catch (err) {
    next(err);
  }
}

module.exports.delete = del;
```
The JavaScript engine will throw an exception if you try to declare a function named delete using a function statement. To work around this, a function named del is declared and then exported as delete.

At this point, you should be able to read and understand the logic. Unlike the previous examples, this HTTP request doesn't have a body, only the id parameter in the route path is used. The "204 No Content" status code is often used with DELETE requests when no response body is sent.

To complete the database logic, return to the db_apis/employees.js file and add the following code to the end.
```
const deleteSql =
 `begin

    delete from job_history
    where employee_id = :employee_id;

    delete from employees
    where employee_id = :employee_id;

    :rowcount := sql%rowcount;

  end;`

async function del(id) {
  const binds = {
    employee_id: id,
    rowcount: {
      dir: oracledb.BIND_OUT,
      type: oracledb.NUMBER
    }
  }
  const result = await database.simpleExecute(deleteSql, binds);

  return result.outBinds.rowcount === 1;
}

module.exports.delete = del;
```
### Parsing JSON Request Bodies
If you look back at the getEmployeeFromRec function in the controller, you'll notice that the request's body property is a JavaScript object. This provides an easy way to get values from the body of the request, but it's not something that happens automatically.

The API you are building, expects clients to send JSON formatted data in the body of POST and PUT requests. In addition, clients should set the Content-Type header of the request to application/json to let the web server know what type of request body is being sent. You can use the built-in express.json middleware to have Express to parse such requests.

Open the services/web-server.js file and add the following lines just below the app.use call that adds morgan to the request pipeline.
```
// Parse incoming JSON requests and revive JSON.
    app.use(express.json({
      reviver: reviveJson
    }));
```
When JSON data is parsed into native JavaScript objects, only the data types supported in JSON will be correctly mapped to JavaScript types. Dates are not supported in JSON and are typically represented as ISO 8601 strings. Using a reviver function, passed to the express.json middleware, you can do the conversions manually. Append the following code to the bottom of the services/web-server.js file.
```
const iso8601RegExp = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d{3})?Z$/;

function reviveJson(key, value) {
  // revive ISO 8601 date strings to instances of Date
  if (typeof value === 'string' && iso8601RegExp.test(value)) {
    return new Date(value);
  } else {
    return value;
  }
}
```
### Testing the API
It's time to test the new CRUD functionality! Up until now, you've been using Firefox to test the API, but this will not work for POST, PUT, and DELETE requests. I'll show you how to how to test the API using curl commands because that's readily available in the VM. However, you might want to look into using a graphical tool, such as Postman (free) or Paw (Mac only, not free).

Start the application and then open another terminal window and execute the following command to create a new employee.
```
curl -X "POST" "http://localhost:3000/api/employees" \
     -i \
     -H 'Content-Type: application/json' \
     -d $'{
  "first_name": "Dan",
  "last_name": "McGhan",
  "email": "dan@fakemail.com",
  "job_id": "IT_PROG",
  "hire_date": "2014-12-14T00:00:00.000Z",
  "phone_number": "123-321-1234"
}'
```
If the request was successful, the response should contain an employee object with an employee_id attribute.




