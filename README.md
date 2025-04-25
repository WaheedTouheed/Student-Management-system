#!/bin/bash

teacher_file="teacher_account.txt"
student_file="student_records.txt"
login_file="login_credentials.txt"
temp_file="temp_records.txt"
grading_criteria=(90 85 80 75 70 65 60 55 50 0)
grading_letters=("A+" "A" "A-" "B+" "B" "B-" "C+" "C" "C-" "F")
max_students=20
initialize_files() {
    [[ ! -f "$teacher_file" ]] && echo "admin:admin123" > "$teacher_file"
    touch "$student_file" "$login_file" "$temp_file"
}
clear_screen() { 
    clear || printf "\033c" || echo -e "\033c"
}

display_header() {
    clear_screen
    echo "============================================"
    echo "    STUDENT MANAGEMENT SYSTEM (SMS)"
    echo "============================================"
}


handle_error() {
    local message=$1
    echo "ERROR: $message"
    sleep 2
    return 1
}

validate_number() {
    [[ "$1" =~ ^[0-9]+$ ]] || handle_error "Invalid number input"
}

validate_marks() {
    validate_number "$1" && [[ "$1" -ge 0 && "$1" -le 100 ]] || handle_error "Marks must be between 0-100"
}

file_exists() {
    [[ -f "$1" ]] || handle_error "File $1 not found"
}


teacher_login() {
    display_header
    echo "TEACHER LOGIN"
    read -p "Username: " username
    read -s -p "Password: " password; echo
    
    file_exists "$teacher_file" || return
    stored_cred=$(grep "^$username:" "$teacher_file" 2>/dev/null | cut -d: -f2)
    
    if [[ "$password" == "$stored_cred" ]]; then
        teacher_menu
    else
        handle_error "Invalid login credentials"
        main_menu
    fi
}

student_login() {
    display_header
    echo "STUDENT LOGIN"
    read -p "Roll Number: " roll_no
    read -s -p "Password: " password; echo
    
    file_exists "$login_file" || return
    stored_cred=$(grep "^$roll_no:" "$login_file" 2>/dev/null | cut -d: -f2)
    
    if [[ "$password" == "$stored_cred" ]]; then
        student_menu "$roll_no"
    else
        handle_error "Invalid login credentials"
        main_menu
    fi
}


calculate_grade() {
    local marks=$1
    for i in "${!grading_criteria[@]}"; do
        [[ "$marks" -ge "${grading_criteria[$i]}" ]] && echo "${grading_letters[$i]}" && return
    done
    echo "F"
}

calculate_cgpa() {
    case $1 in
        "A+") echo "4.0";;
        "A")  echo "4.0";;
        "A-") echo "3.67";;
        "B+") echo "3.33";;
        "B")  echo "3.0";;
        "B-") echo "2.67";;
        "C+") echo "2.33";;
        "C")  echo "2.0";;
        "C-") echo "1.67";;
        *)    echo "0.0";;
    esac
}


add_student() {
    while true; do
        display_header
        echo "ADD NEW STUDENT"
        
        # Check max students limit
        if [[ $(wc -l < "$student_file" 2>/dev/null) -ge $max_students ]]; then
            handle_error "Maximum student limit ($max_students) reached"
            return
        fi

        read -p "Roll Number: " roll_no
        [[ -z "$roll_no" ]] && handle_error "Roll number cannot be empty" && continue
        
        # Check if roll number exists
        if grep -q "^$roll_no:" "$student_file" 2>/dev/null; then
            handle_error "Roll number already exists"
            continue
        fi

        read -p "Full Name: " name
        [[ -z "$name" ]] && handle_error "Name cannot be empty" && continue

        read -p "Marks (0-100): " marks
        validate_marks "$marks" || continue

        grade=$(calculate_grade "$marks")
        cgpa=$(calculate_cgpa "$grade")

        # Add to student file
        echo "$roll_no:$name:$marks:$grade:$cgpa" >> "$student_file" || {
            handle_error "Failed to write to student file"
            return
        }

        # Set password
        while true; do
            read -s -p "Set password: " password; echo
            [[ -z "$password" ]] && handle_error "Password cannot be empty" && continue
            echo "$roll_no:$password" >> "$login_file" || {
                handle_error "Failed to write to login file"
                return
            }
            break
        done

        echo "Student added successfully!"
        read -p "Add another student? (y/n): " choice
        [[ "$choice" != "y" ]] && break
    done
}

update_student() {
    display_header
    echo "UPDATE STUDENT RECORD"
    
    file_exists "$student_file" || return
    
    read -p "Enter Roll Number to update: " roll_no
    [[ -z "$roll_no" ]] && handle_error "Roll number cannot be empty" && return

    record=$(grep "^$roll_no:" "$student_file" 2>/dev/null)
    [[ -z "$record" ]] && handle_error "Student not found" && return

    IFS=: read -r old_roll old_name old_marks old_grade old_cgpa <<< "$record"
    
    echo -e "\nCurrent Details:"
    echo "1. Roll Number: $old_roll"
    echo "2. Name: $old_name"
    echo "3. Marks: $old_marks"
    echo "4. Grade: $old_grade"
    echo "5. CGPA: $old_cgpa"
    
    echo -e "\nWhat would you like to update?"
    echo "1. Name"
    echo "2. Marks"
    echo "3. Cancel"
    
    read -p "Enter choice: " choice
    case $choice in
        1)
            read -p "Enter new name: " new_name
            [[ -z "$new_name" ]] && handle_error "Name cannot be empty" && return
            # Update only the name
            sed -i "/^$roll_no:/s/:$old_name:/:$new_name:/" "$student_file"
            echo "Name updated successfully!"
            ;;
        2)
            read -p "Enter new marks (0-100): " new_marks
            validate_marks "$new_marks" || return
            new_grade=$(calculate_grade "$new_marks")
            new_cgpa=$(calculate_cgpa "$new_grade")
            # Update marks, grade and CGPA
            sed -i "/^$roll_no:/s/:$old_marks:$old_grade:$old_cgpa$/:$new_marks:$new_grade:$new_cgpa/" "$student_file"
            echo "Marks and grades updated successfully!"
            ;;
        3)
            echo "Update cancelled."
            ;;
        *)
            handle_error "Invalid choice"
            ;;
    esac
    sleep 2
}

delete_student() {
    display_header
    echo "DELETE STUDENT RECORD"
    
    file_exists "$student_file" || return
    
    read -p "Enter Roll Number to delete: " roll_no
    [[ -z "$roll_no" ]] && handle_error "Roll number cannot be empty" && return

    if ! grep -q "^$roll_no:" "$student_file"; then
        handle_error "Student not found"
        return
    fi

    # Confirm deletion
    read -p "Are you sure you want to delete this student? (y/n): " confirm
    [[ "$confirm" != "y" ]] && echo "Deletion cancelled." && sleep 1 && return

    # Remove from student file
    grep -v "^$roll_no:" "$student_file" > "$temp_file" && mv "$temp_file" "$student_file"
    
    grep -v "^$roll_no:" "$login_file" > "$temp_file" && mv "$temp_file" "$login_file"
    
    echo "Student deleted successfully!"
    sleep 2
}

view_students() {
    display_header
    echo "STUDENT RECORDS"
    
    file_exists "$student_file" || return
    
    if [[ ! -s "$student_file" ]]; then
        echo "No student records found!"
        sleep 2
        return
    fi

    printf "%-10s %-20s %-10s %-10s %-10s\n" "Roll No" "Name" "Marks" "Grade" "CGPA"
    echo "-----------------------------------------------------"
    awk -F: '{printf "%-10s %-20s %-10s %-10s %-10s\n", $1, $2, $3, $4, $5}' "$student_file"
    read -n 1 -s -r -p "Press any key to continue..."
}

search_student() {
    display_header
    echo "SEARCH STUDENT RECORD"
    
    file_exists "$student_file" || return
    
    read -p "Enter Roll Number: " roll_no
    [[ -z "$roll_no" ]] && handle_error "Roll number cannot be empty" && return

    record=$(grep "^$roll_no:" "$student_file" 2>/dev/null)
    if [[ -z "$record" ]]; then
        handle_error "Student not found"
    else
        IFS=: read -r roll_no name marks grade cgpa <<< "$record"
        echo -e "\nRoll No: $roll_no"
        echo "Name: $name"
        echo "Marks: $marks"
        echo "Grade: $grade"
        echo "CGPA: $cgpa"
    fi
    read -n 1 -s -r -p "Press any key to continue..."
}

list_passed_failed() {
    display_header
    echo "LIST PASSED/FAILED STUDENTS"
    
    file_exists "$student_file" || return
    
    if [[ ! -s "$student_file" ]]; then
        echo "No student records found!"
        sleep 2
        return
    fi

    echo "1. List Passed Students (CGPA >= 2.0)"
    echo "2. List Failed Students (CGPA < 2.0)"
    echo "3. Back to Menu"
    
    read -p "Enter choice: " choice
    case $choice in
        1)
            echo -e "\nPASSED STUDENTS:"
            printf "%-10s %-20s %-10s\n" "Roll No" "Name" "CGPA"
            echo "----------------------------------------"
            awk -F: '$5 >= 2.0 {printf "%-10s %-20s %-10s\n", $1, $2, $5}' "$student_file"
            ;;
        2)
            echo -e "\nFAILED STUDENTS:"
            printf "%-10s %-20s %-10s\n" "Roll No" "Name" "CGPA"
            echo "----------------------------------------"
            awk -F: '$5 < 2.0 {printf "%-10s %-20s %-10s\n", $1, $2, $5}' "$student_file"
            ;;
        3)
            return
            ;;
        *)
            handle_error "Invalid choice"
            ;;
    esac
    read -n 1 -s -r -p "Press any key to continue..."
}

generate_report() {
    display_header
    echo "GENERATE STUDENT REPORT"
    
    file_exists "$student_file" || return
    
    if [[ ! -s "$student_file" ]]; then
        echo "No student records found!"
        sleep 2
        return
    fi

    echo "1. Sort by Roll Number (Ascending)"
    echo "2. Sort by Name (Ascending)"
    echo "3. Sort by CGPA (Descending)"
    echo "4. Sort by Marks (Descending)"
    echo "5. Back to Menu"
    
    read -p "Enter choice: " choice
    case $choice in
        1)
            echo -e "\nSTUDENTS SORTED BY ROLL NUMBER:"
            sort -t: -k1,1n "$student_file" | awk -F: '{printf "%-10s %-20s %-10s %-10s %-10s\n", $1, $2, $3, $4, $5}'
            ;;
        2)
            echo -e "\nSTUDENTS SORTED BY NAME:"
            sort -t: -k2,2 "$student_file" | awk -F: '{printf "%-10s %-20s %-10s %-10s %-10s\n", $1, $2, $3, $4, $5}'
            ;;
        3)
            echo -e "\nSTUDENTS SORTED BY CGPA (DESCENDING):"
            sort -t: -k5,5nr "$student_file" | awk -F: '{printf "%-10s %-20s %-10s %-10s %-10s\n", $1, $2, $3, $4, $5}'
            ;;
        4)
            echo -e "\nSTUDENTS SORTED BY MARKS (DESCENDING):"
            sort -t: -k3,3nr "$student_file" | awk -F: '{printf "%-10s %-20s %-10s %-10s %-10s\n", $1, $2, $3, $4, $5}'
            ;;
        5)
            return
            ;;
        *)
            handle_error "Invalid choice"
            ;;
    esac
    read -n 1 -s -r -p "Press any key to continue..."
}

teacher_menu() {
    while true; do
        display_header
        echo "TEACHER MENU"
        echo "1. Add Student"
        echo "2. View All Students"
        echo "3. Search Student"
        echo "4. Update Student Record"
        echo "5. Delete Student"
        echo "6. List Passed/Failed Students"
        echo "7. Generate Reports"
        echo "8. Logout"
        
        read -p "Enter choice: " choice
        case $choice in
            1) add_student ;;
            2) view_students ;;
            3) search_student ;;
            4) update_student ;;
            5) delete_student ;;
            6) list_passed_failed ;;
            7) generate_report ;;
            8) main_menu ;;
            *) handle_error "Invalid choice"; sleep 1 ;;
        esac
    done
}


student_menu() {
    local roll_no=$1
    file_exists "$student_file" || return
    
    record=$(grep "^$roll_no:" "$student_file" 2>/dev/null)
    [[ -z "$record" ]] && handle_error "Student record not found" && main_menu
    
    IFS=: read -r roll_no name marks grade cgpa <<< "$record"
    
    while true; do
        display_header
        echo "STUDENT MENU ($name)"
        echo "1. View Grades"
        echo "2. View CGPA"
        echo "3. View Full Details"
        echo "4. Logout"
        
        read -p "Enter choice: " choice
        case $choice in
            1) 
                echo -e "\nMarks: $marks"
                echo "Grade: $grade"
                sleep 3
                ;;
            2)
                echo -e "\nCGPA: $cgpa"
                sleep 3
                ;;
            3)
                echo -e "\nRoll No: $roll_no"
                echo "Name: $name"
                echo "Marks: $marks"
                echo "Grade: $grade"
                echo "CGPA: $cgpa"
                read -n 1 -s -r -p "Press any key to continue..."
                ;;
            4) 
                main_menu
                ;;
            *)
                handle_error "Invalid choice"
                sleep 1
                ;;
        esac
    done
}

main_menu() {
    while true; do
        display_header
        echo "MAIN MENU"
        echo "1. Teacher Login"
        echo "2. Student Login"
        echo "3. Exit System"
        
        read -p "Enter choice: " choice
        case $choice in
            1) teacher_login ;;
            2) student_login ;;
            3) 
                clear_screen
                echo "Thank you for using the Student Management System"
                exit 0
                ;;
            *) 
                handle_error "Invalid choice"
                sleep 1
                ;;
        esac
    done
}

trap 'echo -e "\nScript interrupted. Exiting safely..."; exit 1' INT TERM
initialize_files
main_menu

