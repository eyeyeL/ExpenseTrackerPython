# ExpenseTracker
# Customized expense tracker, which also allows user to keep track of expense shared with other people! In addition, automatically create expenses that are recurring every month.


#Connect to database
import mysql.connector
mydb = mysql.connector.connect(host='localhost',database='testdb',user='root',password='', autocommit=True)
mycursor = mydb.cursor()
import datetime                                                          #datetime module
import calendar                                                          #calendar module
from prettytable import PrettyTable                                      #import pretty table module for command line data table

class Expense:
    def __init__(self):
        #set each variable to empty string or "0" if variable is an integer
        self.store = ""
        self.expense_cat = ""
        self.amount = 0
        self.date = 0
        self.shared = False
        self.shared_number = 0                                           #shared_number = number of people shared with
        self.percent_shared = 0
        self.recurring_charge = False
        self.name = ""
        #editted 23/6/2020: add user_id
        self.user_id = ""

    #calculate the amount shared with other people
    def calc_amount_per_person(self):
        divider_decimal = self.percent_shared / 100                      #calculating divider into decimal from percentage input by user
        return round(self.amount * divider_decimal, 2)                   #returning amount shared among people based on the percentage input by the user
        

    #get store ID from mysql
    def get_store_id(self):
        mycursor = mydb.cursor()
        args = [self.store, 0]
        result = mycursor.callproc("get_store_id", args)                  #calling stored procedure in mySQL
        mycursor.close()
        return result[1]


    #get category ID from mysql
    def get_cat_id(self):
        mycursor = mydb.cursor()
        args = [self.expense_cat, 0]
        result = mycursor.callproc("get_category_id", args)              #calling stored procedure in mySQL
        mycursor.close()
        return result[1]
        
    
    #store the details of the shared expense 
    def shared_person(self, amount_shared, store_id):
        #user can input more than one name
        for person in self.name:                                         #iterates through the list and stores the same expense for different name
            mycursor = mydb.cursor()
            args = [person, self.date, amount_shared, store_id]
            mycursor.callproc("shared_amount_person", args)              #calling stored procedure in mySQL to store the expense for different names inputted
            mycursor.close()



    #store user input expense data into the database
    def store_data(self, store, expense_cat, amount, date, user_id, shared = False, shared_number = 0, percent_shared = 0, recurring_charge = False, name = ""):
        self.store = store.lower()
        self.expense_cat = expense_cat.lower()
        self.amount = round(amount, 2)
        self.date = date
        self.shared = shared
        self.shared_number = shared_number                               #shared_number = number of people shared with
        self.percent_shared = round(percent_shared, 2)
        self.recurring_charge = recurring_charge
        self.name = name
        self.user_id = user_id

        new_store_id = self.get_store_id()                               #getting stored id by calling get_store_id method
        new_cat_id = self.get_cat_id()                                   #getting expense category id by calling get_cat_id method

        if self.shared == True:                                          #if the expense the user input is shared with another person
            amount_shared = self.calc_amount_per_person()                #getting the amount shared with the person by calling calc_amount_per_person method
            self.shared_person(amount_shared, new_store_id)              #calling the shared_person method to store shared_expense information 
        else:
            amount_shared = 0                                            #value is 0 when expense is not shared

        mycursor = mydb.cursor()

        args = [new_store_id, self.amount, self.shared, self.shared_number, self.percent_shared, new_cat_id, self.recurring_charge, amount_shared, self.date, self.user_id]
        mycursor.callproc("store_new_expense", args)                     #calling stored procedure to store the expense inputted by user
        mycursor.close()
        return "Data Stored"


    #getting the monthly report
    #edit 22/6/2020: add days so range can be set

    def monthly_report(self, start_range, end_range, user_id, leap_year = False):
        self.start_date = start_range
        self.end_date = end_range
        

        mycursor = mydb.cursor()
        args = [self.start_date, self.end_date, user_id]

        mycursor.callproc("month_expense", args)                         #calling stored procedure in mySQL to get the montly data 
        
        print("")
        print("Monthly Report")

        #using a module to print out the rows in a data table
        print_table = PrettyTable(["Date", "Store Name", "Category", "Amount", "Shared Expense", "Percent Shared", "Recurring Charge", "Shared Amount", "Name of Person"])


        for result in mycursor.stored_results():                         #looping through the results stored in mycursor
            list_result = result.fetchall()                              #stored the results (a list) into a new variable in the case of multiple rows
        
        total_amount = 0                                                 #sums up the total amount for the time range called by user
        total_shared_amount = 0                                          #total amount that is shared with others
        name_shared_person = []                                          #a list of name(s) of the person shared, used for printing out the total amount shared with each person at the end of report

        #variables current and previous: allows to print only rows that are not repeated due to the shared expense with multiple people
        current_store = ""
        previous_store = ""
        current_amount = 0
        previous_amount = 0
        current_date = ""
        previous_date = ""
        current_category = ""
        previous_category = ""

        for tup in list_result:                                          #looping through the list of results to print out individual row
            current_date = tup[0]
            current_store = tup[1]
            current_category = tup[2]
            current_amount = tup[3]
 
            #Conditional: to set shared value to Yes or No
            if tup[4] == 1:
                shared = "Yes"
                name = tup[8]
                if tup[8] not in name_shared_person:
                    name_shared_person.append(tup[8])
            else:
                #shared = "No     "
                #name = "            "
                shared = "No"
                name = ""

            #conditional: to check if the row is a repeat of the previous row due to shared expenses with mulitple people
            if current_date == previous_date and current_store == previous_store and current_category == previous_category and current_amount == previous_amount:
                print_status = 0                                           #if the row is a repeated row, print_status is 0 meaning, it won't print
                pass
            else:
                total_amount += current_amount                             #sum of total amount
                print_status = 1
                #Conditional: if the expense is a shared value, the shared_amount is added to the total of the shared amount
                if tup[4] == 1:
                    total_shared_amount += tup[3]

            #set the "previous" variables using the "current" variables"
            previous_date = current_date
            previous_store = current_store
            previous_category = current_category
            previous_amount = current_amount

            #Conditional: if the expense is recurring, set to Yes or No
            if tup [6] == 1:
                recurring = "Yes"
            else:
                recurring = "No"
            
            #Condional: check prints status, if print status is true, the values are added to the print_table
            if print_status == 1:
                print_table.add_row([tup[0], tup[1], tup[2], tup[3], shared, tup[5], recurring,tup[7], name])

        print(print_table)
        print("")
        print("")
        print("Total expense: " + str(round(total_amount,2)))

        #printing the sum by categories
        sum_by_cat = self.get_sum_by_category(self.start_date,self.end_date)
        for category in sum_by_cat:
            print("Total expense for " + category[0] + ": " + str(round(category[1],2)))
                   
        print("Total shared expense: " + str(round(total_shared_amount,2)))

        
        mycursor.close()

        #editted 25/6/2020: call procedure for summing all expensed shared by others
        mycursor = mydb.cursor()
        args = [self.start_date, self.end_date, user_id]

        mycursor.callproc("expense_from_others", args)                         #calling stored procedure in mySQL to get shared value that is marked by user's name

        for result in mycursor.stored_results():
            sum_result = result.fetchall()

        mycursor.close()

        print("Monthly expense shared with you: " + str(round(sum_result[0][0],2)))

        for name in name_shared_person:
            total_per_person = 0
            for tup in list_result:
                if name == tup[8]:
                    total_per_person += tup[7]

            print("Amount per person with " + name + " : " + str(round(total_per_person,2)))
        
        


    #added 27/5/2020: fetching all the expense categories in a list
    def fetch_expense_categories(self):
        categories_with_id = {}
        mycursor = mydb.cursor()
        mycursor.callproc("fetch_expense_categories")
        
        for result in mycursor.stored_results():
            list_cat = result.fetchall()

        for item in list_cat:
            categories_with_id[item[1]] = item[0]

        mycursor.close()

        return categories_with_id

    #added 27/5/2020: getting the sum of the particular expense category
    def get_sum_by_category(self, start_date, end_date):
        cat_dic = self.fetch_expense_categories()                       #fetch all categories

        expense_by_cat = []                                             #list of of tuples

        for cat_id in cat_dic:
            mycursor = mydb.cursor()
            args = [start_date, end_date, cat_id, 0]
            result = mycursor.callproc("get_sum_by_category", args)

            #if there is None is return for a particular value, the category is passed and not stored in the expense by category
            if result[3] == None:
                pass
            else:
                expense_by_cat.append((cat_dic[cat_id], round(result[3], 2)))

            mycursor.close()

        return expense_by_cat

    #added 22/5/2020: changing the amount on the recurring charge
    def change_amount_recurring_charge(self, date, store, old_amount, new_amount, category):
        #date = date of the charge the user want to edit
        #old_amount = the original amount of the recurring charge
        #new_amount = the new amount the user want to change to
        #category = category of the charge

        mycursor = mydb.cursor()

        args = [date, store, old_amount, new_amount, category, 0]
        result = mycursor.callproc("change_amount_recurring_charge", args)
        mycursor.close()  

        return result[5]                                               #returned results would either be if the charge exist or not


    #added 22/5/2020: changing the date on the recurring chare
    def change_date_recurring_charge(self, old_date, new_date, store, old_amount, category):
        #old_date = original date of the charge
        #new_date = the new date the recurring charge would be changed to 
        #store = name of the store of the charge
        #old_amount = the original amount of recurring charge
        #category = cateogory of the charge

        mycursor = mydb.cursor()

        args = [old_date, new_date, store, old_amount, category, 0]
        result = mycursor.callproc("change_date_recurring_charge", args)
        mycursor.close()

        return result[5]                                                #returned results would either be if the charge exist or not

    #added 27/5/2020: deleting a range of recurring charges
    def delete_recurring_charge(self, start_date, end_date, store, amount, category):
        #start_date = start of the date range the user want the charge to be deleted
        #end_date = end of the date range the user want the charge to be deleted
        #store = name of the store
        #amount = amount of the charge
        # category = category of the charge

        mycursor = mydb.cursor()
        args = [start_date, end_date, store, amount, category]
        mycursor.callproc("look_up_rows", args)

        for result in mycursor.stored_results():
            list_results = result.fetchall()
        mycursor.close()

        for row_id in list_results:
            mycursor = mydb.cursor()
            args = [row_id[0]]
            mycursor.callproc("delete_recurring_charge",args)
            mycursor.close()

        return "Your recurring charges have been deleted."
        

#Command LINE first user input: choosing between inputting new expense or looking up monthly expense
print("Welcome to your Expense Tracker!")
#editted 23/6/2020: adding user_id, shortened typing
user_id = input("Input your username: ")

big_status = 0                                                         #while loop runs until user pick one of the four choices, will prompt user to choose again if choice not recognize
while big_status == 0:
    #editted 23/6/2020: added change user
    print("Welcome " + user_id.upper() + " to your Expense Tracker!") 
    user_choice = input("Would you like to input new expense (type: NE), look up monthly spending (type: M), edit recurring charge (type: E), or change user (type: C) ? ")
    if user_choice == "NE" or user_choice == "ne":                                   #first choice: inputting new expense
        big_status = 1
        print("Please input the following information to log your new expense:")

        #prompt user to input the following variables
        store = input("Store: ")                                                    #name of store
        expense_cat = input("Expense category: ")
        amount = float(input("Amount spent: "))

        year = int(input("Year of the expense (YYYY): "))
        month = int(input("Month of the expense (M): "))
        day = int(input("Day of the expense (D): "))
        date = datetime.date(year, month, day)                                     #date from user's inputted data

        recurring = input("Is this a recurring charge? (Y or N): ")
        recurring_months = 0
        status = 0
        while status == 0:                                            #while loop runs until user picked the correct choice

            if recurring == "Y" or recurring == "y":
                recurring_months = int(input("How many months (> 0) for recurring charge? "))       #added 22/5/2020: use the number to auto-generate new expense row (eg: 1 means 1 additional month)
                recurring = True
                status = 1
            elif recurring == "N" or recurring == "n":
                recurring = False
                status = 1
            else:
                recurring = input("Wrong input, please type Y or N:")
        
                
        shared_expense = input("Was this a shared expense? (Y or N): ")
        status = 0
        while status == 0:                                             #while loop runs until user picked the correct choice: shared expense or not

            if shared_expense == "Y" or shared_expense == "y":
                status = 1
                shared_expense = True
                shared_num = int(input("How many people was this expense shared with, including yourself? "))
                #editted 21/7/2020: automatically add shared percentage
                percent_shared = 100 / shared_num

                name = []
                
                #loop to prompt user to add the name(s) of persons the charged was shared with
                for n in range(0, shared_num - 1):
                    input_name = input("Name of the person: ")
                    name.append(input_name)
            elif shared_expense == "N" or shared_expense == "n":
                status = 1
                shared_expense = False
                shared_num = 0
                percent_shared = 0
                name = []
            else:
                shared_expense = input("Wrong input, please type Y or N: ")
        
        #editted 23/6/2020: add user_id
        new_expense = Expense()                                      #calling the expense class
        print(new_expense.store_data(store, expense_cat, amount, date, user_id, shared_expense, shared_num, percent_shared, recurring, name))
                                       
        number_charges = recurring_months                            #added 22/5/2020: adding a loop to insert the recurring charge for addtional # of months indicated by user
        while number_charges > 0:
            if month == 12:
                month = 1
                year += 1
            else:
                month += 1
            date = datetime.date(year, month, day)
            #editted 23/6/2020: add username
            new_expense.store_data(store, expense_cat, amount, date, user_id, shared_expense, shared_num, percent_shared, recurring, name)
            number_charges -= 1

        more_action = input("Would you like to do additional transaction? Type Y or N: ")       #prompt uswer for additional transaction
        
        if more_action == "Y" or more_action == "y":
            big_status = 0
        elif more_action == "N" or more_action == "n":
            big_status = 1
            print("Thank you!")


    #second choice: monthly report
    elif user_choice == "M" or user_choice == "m":
        big_status = 1

        #editted 22/6/2020: adding a range for the monthly report
        year = int(input("Which expense year would you like to see (YYYY)? "))
        month_start = int(input("Which expense month for start of range (M)? "))
        day_start = int(input("Which expense day for the start of range (D)? "))

        month_end = int(input("Which expense month for end of range (M)? "))
        day_end = int(input("Which expense day for the end of range (D)? "))

        leap = calendar.isleap(year)                                #editted 27/5/2020: delete the leap year loop and just use calendar module

        start_range = datetime.date(year, month_start, day_start)
        end_range = datetime.date(year, month_end, day_end)
        
        new_expense = Expense()                                     #calling the expense class
        #editted 23/6/2020: add user id check
        new_expense.monthly_report(start_range, end_range, user_id, leap)

        more_action = input("Would you like to do additional transaction? Type Y or N: ")
        
        if more_action == "Y" or more_action == "y":
            big_status = 0
        elif more_action == "N" or more_action == "n":
            print("Thank you!")
            big_status = 1
    
    #third choice: edit the recurring charge: added 22/5/2020
    elif user_choice == "E" or user_choice == "e":
        big_status = 1

        change_delete_status = input("Would you like to edit (type: edit) or delete (type: delete) the recurring charge? ")
        store_name = (input("Please enter store name: ")).lower()
        old_amount = int(input("What is the current amount? "))
        category_name = input("What is the category of the charge? ")

        change_expense = Expense()

        change_status = 0
        while change_status == 0:
            if change_delete_status == "edit":
                change_status = 1

                year = int(input("Please input the year (YYYY): "))
                month = int(input("Please input the month (M): "))
                day = int(input("Please input the day (D): "))
                old_date = datetime.date(year, month, day)

                change_what = input("Would you like to change the amount (type: amount) or date (type: date)? ")

                edit_status = 0

                while edit_status == 0:
                    if change_what == "amount":
                        edit_status = 1
                        new_amount = float(input("What is the new amount you would like to change it to? "))

                        print(change_expense.change_amount_recurring_charge(old_date, store_name, old_amount, new_amount, category_name))
                    elif change_what == "date":
                        edit_status = 1
                        new_year = int(input("What is the new year you would like to change to? (YYYY) "))
                        new_month = int(input("What is the new month you would like to change to? (M) "))
                        new_day = int(input("What is the new day you would like to change to? (D) "))

                        new_date = datetime.date(new_year, new_month, new_day)

                        print(change_expense.change_date_recurring_charge(old_date, new_date, store_name, old_amount, category_name))
                    else:
                        change_what = input("Choice not recognize. Please choose between 'amount' or 'date': ")
            elif change_delete_status == "delete":
                change_status = 1

                #User enter the start date for the range to delete
                print("\n"+"Enter the date for the start of the range:")
                delete_start_year = int(input("Enter the year: "))
                delete_start_month = int(input("Enter the month: "))
                delete_start_day = int(input("Enter the day: "))

                delete_start_date = datetime.date(delete_start_year, delete_start_month, delete_start_day)

                #User enter the end date for the range to delete
                print("\n" + "Enter the date for the end of the range")
                delete_end_year = int(input("Enter the year: "))
                delete_end_month = int(input("Enter the month: "))
                delete_end_day = int(input("Endter the day: "))

                delete_end_date = datetime.date(delete_end_year, delete_end_month, delete_end_day)

                print(change_expense.delete_recurring_charge(delete_start_date, delete_end_date, store_name, old_amount, category_name))
            else:
                change_delete_status = input("Invalid choice: Please indicate if you want to (edit) or (delete) the recurring charge(s). ")
            
        more_action = input("Would you like to do additional transaction? Type Y or N: ")
        
        if more_action == "Y" or more_action == "y":
            big_status = 0
        elif more_action == "N" or more_action == "n":
            print("Thank you!")
            big_status = 1

    #editted 23/6/2020: add a choice for user to change the username
    elif user_choice == "C" or user_choice == "c":
        user_id = input("Please insert a new username: ")
    #prompt user to choose again
    else:
        print("Choice not recognize")

