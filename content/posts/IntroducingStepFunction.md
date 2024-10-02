+++
title = 'Introduction To Step Function: Orchestrating Complex Workflows'
date = 2024-10-03T00:05:35+05:30
+++

I was inspired to replicate the functionality of AWS Step Functions in my own code when I couldn't find any libraries that suited my needs for orchestrating complex workflows. This led me to design a custom Step Function library to handle sequential and parallel execution, branch logic, error handling, and more.

## What is a Step Function?

A **Step Function** helps developers orchestrate complex workflows and track the execution status at each step. It essentially functions as a **directed acyclic graph (DAG)** where each node represents a function call, and the connections between nodes define the order of execution. Every node in the graph points to the next step, unidirectionally, and each step depends on the result of the previous step—except for the first.

You can install the package directly from [PyPI](https://pypi.org/project/stepfunction/).

```bash
pip install stepfunction
```

## Example

The below example handles car purchase transactions by validating the transaction, updating shipping, payment, and inventory records in parallel, and notifying the customer. If any step fails, it gracefully triggers a fallback to manage the error without halting the workflow.

{{< figure src="/images/Validate_Car_Purchase_Transaction_Workflow.gv.cairo.png" caption="Validate Car Purchase Transaction Workflow" >}}


```python
from stepfunction.core.step_function import StepFunction

# Simulating the workflow steps for handling a car purchase transaction

def validate_car_purchase_transaction():
    """Simulate validating the car purchase transaction."""
    pass

def check_if_car_purchase_transaction_is_valid():
    """Simulate checking if the transaction details are valid."""
    pass

def send_notification_to_user_email():
    """Simulate sending a notification to the user via email."""
    pass

def update_car_purchase_transaction_status():
    """Simulate updating the status of the car purchase transaction."""
    pass

def update_shipping_db_record():
    """Simulate updating the shipping database record."""
    pass

def update_payment_db_record():
    """Simulate updating the payment database record."""
    pass

def update_inventory_db_record():
    """Simulate updating the inventory database record."""
    pass

def end_step():
    """Simulate the final step to conclude the workflow."""
    pass

def generic_failure_handler():
    """Simulate handling a generic failure during the workflow."""
    pass



async def validate_account_workflow():
    step_function = StepFunction(
        name="Validate_Car_Purchase_Transaction_Workflow")

    step_function.add_step("Parse_Car_Purchase_Transaction", func=validate_car_purchase_transaction,
                           next_step="Check_If_Car_Purchase_Transaction_Is_Valid", on_failure="Generic_Failure_Handler")

    step_function.add_step("Check_If_Car_Purchase_Transaction_Is_Valid", func=check_if_car_purchase_transaction_is_valid, branch={
        "valid": "Update_Company_DB_Records",
        "invalid": "Send_Notification_To_User_Email"
    }, on_failure="Generic_Failure_Handler")

    # Parallel validation steps
    step_function.add_step(
        "Update_Company_DB_Records",
        {
            "update_shipping_db_record": update_shipping_db_record,
            "update_payment_db_record": update_payment_db_record,
            "update_inventory_db_record": update_inventory_db_record
        },
        next_step="Update_Car_Purchase_Transaction_Status",
        parallel=True,
        stop_on_failure=False
    )

    step_function.add_step("Update_Car_Purchase_Transaction_Status",
                           func=update_car_purchase_transaction_status, next_step="Send_Notification_To_User_Email", on_failure="Generic_Failure_Handler")

    step_function.add_step("Send_Notification_To_User_Email",
                           func=send_notification_to_user_email, next_step="End", on_failure="Generic_Failure_Handler")

    step_function.add_step("End", func=end_step)

    step_function.add_step("Generic_Failure_Handler",
                           func=generic_failure_handler)

    await step_function.execute()

    step_function.visualize()
```

## Key Features of the Step Function Library

### 1. Sequential and Parallel Execution
- The library supports both **sequential** execution, where each step follows the previous one, and **parallel** execution, where multiple steps can run concurrently.

### 2. Error Handling and Branching
- Each step can specify an `on_failure` handler, which is executed when a step fails. You can also define branching logic that allows a step to branch to different steps based on its result.

### 3. Context Management
- The library keeps track of the context of each step, storing the results or exceptions encountered during execution. This context can be retrieved for further analysis.

### 4. Sub-Step Functions
- **Sub-step functions** allow you to nest workflows within workflows, adding modularity and flexibility to your design. This is particularly useful when you have reusable workflows that can be applied in different contexts. By breaking larger workflows into smaller sub-step functions, you enhance code readability and maintainability.

## Anatomy of a Step

Each step in the workflow is defined with the following parameters:

| **Argument**       | **Purpose**                                                           | **Type**                                        |
|--------------------|-----------------------------------------------------------------------|-------------------------------------------------|
| `name`             | Name of the step                                                      | `str`                                           |
| `func`             | Function to be executed                                               | `Union[Callable[[Any], Any], Dict[str, Callable[[Any], Any]]]` |
| `next_step`        | Name of the next step to be executed (if no branching)                 | `Optional[str]`                                 |
| `on_failure`       | Name of the step to execute if the current step fails                  | `Optional[str]`                                 |
| `branch`           | Logic to decide the next step based on the result of the current step  | `Optional[Dict[Any, str]]`                      |
| `parallel`         | Whether to execute the step in parallel with other steps               | `bool` (default: `False`)                       |
| `stop_on_failure`  | Whether to stop all parallel executions if one step fails              | `bool` (default: `False`)                       |

This flexible structure makes it easy to define workflows that can handle different execution paths and failure scenarios.

## Key Properties of the Step Function

Once a Step Function is defined, you can access various properties during workflow execution:

| **Property**      | **Purpose**                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `name`            | The name of the Step Function                                                |
| `steps`           | A dictionary containing all the steps defined in the workflow                |
| `last_result`     | The result of the most recently executed step                                |
| `context`         | A dictionary that stores the results of all steps, including exceptions      |
| `status`          | The current status of the Step Function (e.g., `INITIALIZED`, `RUNNING`, `COMPLETED`, `FAILED`) |


