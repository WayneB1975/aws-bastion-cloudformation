# EC2 Instance Name Generator 🖥️

A Python script that automatically generates unique EC2 instance names for approved departments.

---

## What It Does

- Prompts the user for their department name and how many instance names they need
- Validates that only approved departments can generate names
- Generates unique random names by combining letters and numbers
- Handles incorrect uppercase/lowercase input from the user

---

## Approved Departments

Only the following departments are authorized to use this tool:

- Marketing
- Accounting
- FinOps

---

## Example Output

```
How many EC2 instance names do you need?: 3
Enter your department's name: Accounting
Accounting-A3GD17
Accounting-B2FC45
Accounting-G1DA36
```

If an unauthorized department tries to use it:

```
How many EC2 instance names do you need?: 3
Enter your department's name: Engineering
Your Department is not authorized to use this Name Generator
```

---

## How to Run

1. Make sure Python is installed on your machine
2. Clone this repository:
```
git clone https://github.com/YourUsername/ec2-name-generator
```
3. Navigate to the project folder and run:
```
python randon_name_generator.py
```

---

## What I Learned

- Writing and calling Python functions
- Working with loops and string manipulation
- Input validation and handling user errors
- Using `.lower()` to handle case-insensitive input

---

## Technologies Used

- Python 3
- `random` module

---

## Author

**Duane Williams**  
[LinkedIn](https://https://www.linkedin.com/in/duane-williams-/) | [GitHub](https://github.com/WayneB1975/EC2-Name-Generator)
