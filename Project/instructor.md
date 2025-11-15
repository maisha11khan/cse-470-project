from flask import Flask, render_template, request, redirect, url_for, session, flash, jsonify
from flask_wtf import FlaskForm
from flask_wtf.file import FileRequired, FileAllowed
from wtforms import StringField, PasswordField, SubmitField, DateField, FileField
from wtforms.validators import DataRequired, Email, ValidationError
from werkzeug.utils import secure_filename
import bcrypt
import os
from flask_mysqldb import MySQL

# Create an instance of the Flask class
app = Flask(__name__)

# MySQL Configuration
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'root'
app.config['MYSQL_PASSWORD'] = ''
app.config['MYSQL_DB'] = 'project_ols'
app.secret_key = 'your_secret_key_here'

mysql = MySQL(app)

class RegisterForm(FlaskForm):
    name = StringField("Name",validators=[DataRequired()])
    phone = StringField("Phone No",validators=[DataRequired()])
    email = StringField("Email",validators=[DataRequired(), Email()])
    password = PasswordField("Password",validators=[DataRequired()])
    course_name = StringField("Course Name",validators=[DataRequired()])
    address = StringField("Address",validators=[DataRequired()])
    dob = DateField("Date of Birth", format='%Y-%m-%d', validators=[DataRequired()])
    bg = StringField("Blood Group",validators=[DataRequired()])
    profile_picture = FileField("Profile Picture", validators=[FileRequired(), FileAllowed(['jpg', 'jpeg'], 'Images only (.jpg, .jpeg)')])
    submit = SubmitField("Register")
    
class LoginForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired(), Email()])
    password = PasswordField("Password", validators=[DataRequired()])
    submit = SubmitField("Login")

# Home route 
@app.route('/')
def index():
    return render_template('i_index.html')

@app.route('/i_register', methods=['GET', 'POST'])
def i_register():
    form = RegisterForm()

    if form.validate_on_submit():
        name = form.name.data
        phone = form.phone.data
        email = form.email.data
        password = form.password.data
        course_name = form.course_name.data
        address = form.address.data
        dob = form.dob.data
        bg = form.bg.data
        profile_picture = form.profile_picture.data  

        # Handle the profile picture file upload
        if profile_picture:
            filename = secure_filename(profile_picture.filename)  
            file_path = os.path.join('static/uploads', filename) 
            file_path = file_path.replace("\\", "/")

            profile_picture.save(file_path)
            
        else:
            file_path = None  

        # Store the user data in the database
        cursor = mysql.connection.cursor()
        cursor.execute("INSERT INTO instructors (name, phone_no, email, password, course_name, address, date_of_birth, blood_group, profile_picture) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)", 
                       (name, phone, email, password, course_name, address, dob, bg, file_path))
        mysql.connection.commit()
        cursor.close()

        return redirect(url_for('i_login'))  # Redirect to login after successful registration

    return render_template('i_register.html', form=form)

@app.route('/i_login', methods=['GET', 'POST'])
def i_login():
    form = LoginForm()
    if form.validate_on_submit():
        email = form.email.data
        password = form.password.data

        # Fetch student from the database using email
        cursor = mysql.connection.cursor()
        cursor.execute("SELECT * FROM instructors WHERE email=%s", (email,))
        instructor = cursor.fetchone()
        cursor.close()

        # Check if student exists and if the password matches directly (without bcrypt)
        if instructor and password == instructor[4]:  # instructor[4] is the password field in the database
            session['instructor_id'] = instructor[1]  # Store instructor_id in session
            session['instructor_name'] = instructor[0]
            return redirect(url_for('instructor_dashboard'))
        else:
            flash("Login failed. Please check your email and password")
            return redirect(url_for('i_login'))

    return render_template('i_login.html', form=form)

@app.route('/instructor_dashboard')
def instructor_dashboard():
    
    if 'instructor_id' in session:
        instructor_id = session['instructor_id']

        cursor = mysql.connection.cursor()
        cursor.execute("SELECT * FROM instructors where instructor_id=%s", (instructor_id,))
        instructor = cursor.fetchone()
        cursor.close()
           
        # Fetch courses for the instructor
        cursor = mysql.connection.cursor()
        cursor.execute("SELECT course_code, course_name, course_picture FROM courses WHERE instructor_id=%s", (instructor_id,))
        courses = cursor.fetchall()
        cursor.close()
        
        if instructor:
            instructor_name = instructor[0]  # The first column is the instructor's name
            no_courses = len(courses) == 0
            return render_template(
                'instructor_dashboard.html', 
                instructor=instructor, 
                instructor_name=instructor_name, 
                courses=courses,
                no_courses=no_courses
            )
                
    return redirect(url_for('i_login'))

@app.route('/add_new_course', methods=['GET', 'POST'])
def add_new_course():
    if 'instructor_id' not in session:
        return redirect(url_for('i_login'))

    message = None
    course_code = None
    if request.method == 'POST':
        instructor_id = session['instructor_id']
        course_name = request.form['course_name']
        category = request.form['course_category']
        course_picture = request.files['course_picture']
        course_exam = request.form['course_exam']

        picture_path = f"static/uploads/{course_picture.filename}"
        course_picture.save(picture_path)

        cursor = mysql.connection.cursor()
        cursor.execute("""
            INSERT INTO courses (instructor_id, course_name, category, num_of_module, num_of_students, completation_rate, course_picture, course_exam)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """, (instructor_id, course_name, category, 0, 0, 0.00, picture_path, course_exam))
        mysql.connection.commit()
        
        # Fetch the course_code of the newly created course
        cursor.execute("SELECT course_code FROM courses WHERE course_name = %s AND instructor_id = %s", (course_name, instructor_id))
        course_code = cursor.fetchone()[0]
        message = "Successfully created the course!"

    return render_template('add_new_course.html', message=message, course_code=course_code)


@app.route('/create_module', methods=['POST'])
def create_module():
    if 'instructor_id' not in session:
        return redirect(url_for('i_login'))

    course_code = request.form['course_code']
    outline_pdf = request.files.get('outline_pdf')
    module_video = request.form['module_video']
    assignment = request.files['assignment']
    deadline = request.form['deadline']
    module_content = request.files['module_content']

    cursor = mysql.connection.cursor()

    # Increment module count
    cursor.execute("SELECT num_of_module FROM courses WHERE course_code=%s", (course_code,))
    current_modules = cursor.fetchone()[0]
    new_module_no = current_modules + 1

    # Save files if provided
    outline_path = None
    if outline_pdf:
        outline_path = f"{outline_pdf.filename}"
        outline_pdf.save(outline_path)

    assignment_path = f"{assignment.filename}"
    assignment.save(assignment_path)

    module_content_path = f"{module_content.filename}"
    module_content.save(module_content_path)

    # Insert the module data into the content table
    cursor.execute("""
        INSERT INTO content (course_code, module_no, outline_pdf, module_video, assignment, deadline, pdf_file)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
    """, (course_code, new_module_no, outline_path, module_video, assignment_path, deadline, module_content_path))
    
    # Update num_of_module for the course
    cursor.execute("""
        UPDATE courses SET num_of_module = num_of_module + 1 WHERE course_code = %s
    """, (course_code,))
    
    mysql.connection.commit()
    return jsonify({'success': True})

    #return render_template('add_new_course.html', module_message="Successfully created the module for the course!")    

@app.route('/get_courses', methods=['GET', 'POST'])
def get_courses():
    if request.method == 'POST':
        data = request.get_json()
        query = data.get('query', None)
        filter_category = data.get('filter', None)
    else:
        # Handle GET request (optional)
        query = None
        filter_category = None

    cursor = mysql.connection.cursor()
    if query and filter_category:
        cursor.execute(
            "SELECT c.course_code, c.course_name, c.course_picture, i.name AS instructor_name "
            "FROM courses c "
            "JOIN instructors i ON c.instructor_id = i.instructor_id "
            "WHERE c.course_name LIKE %s AND c.category = %s",
            (f"%{query}%", filter_category,)
        )
    elif query:
        cursor.execute(
            "SELECT c.course_code, c.course_name, c.course_picture, i.name AS instructor_name "
            "FROM courses c "
            "JOIN instructors i ON c.instructor_id = i.instructor_id "
            "WHERE c.course_name LIKE %s",
            (f"%{query}%",)
        )
    elif filter_category:
        cursor.execute(
            "SELECT c.course_code, c.course_name, c.course_picture, i.name AS instructor_name "
            "FROM courses c "
            "JOIN instructors i ON c.instructor_id = i.instructor_id "
            "WHERE c.category = %s",
            (filter_category,)
        )
    else:
        cursor.execute(
            "SELECT c.course_code, c.course_name, c.course_picture, i.name AS instructor_name "
            "FROM courses c "
            "JOIN instructors i ON c.instructor_id = i.instructor_id"
        )

    courses = cursor.fetchall()
    cursor.close()

    course_list = []
    for course in courses:
        course_list.append({
            'course_code': course[0],
            'course_name': course[1],
            'course_picture': course[2],
            'instructor_name': course[3],
            'description': "In order to extract knowledge and insights from both organized and unstructured data, data science is an interdisciplinary profession."
        })

    # For GET request, you may render an HTML page if needed
    if request.method == 'GET':
        return render_template('display_courses.html', courses=course_list)
    
    return {'data': course_list}


# Profile Page Route
@app.route('/i_profile', methods=['GET', 'POST'])
def i_profile():
    if 'instructor_id' not in session:
        flash("You need to login first!", 'danger')
        return redirect(url_for('i_login'))
    
    instructor_id = session['instructor_id']
    cursor = mysql.connection.cursor()
    
    # If the form is submitted (POST request), update the student's information
    if request.method == 'POST':
        # Get updated data from the form
        new_name = request.form['name']
        new_password = request.form['password']
        new_course_name = request.form['course_name']
        new_address = request.form['address']
        new_dob = request.form['date_of_birth']
        new_blood_group = request.form['blood_group']
        new_picture = request.files['profile_picture']
        
        # Fetch instructor data from the database
        cursor = mysql.connection.cursor()
        cursor.execute("SELECT * FROM instructors WHERE instructor_id = %s", (instructor_id,))
        instructor = cursor.fetchone()  # Fetch the instructor's record
        cursor.close()
        
        if not instructor:
            flash("Instructor not found", 'danger')
            return redirect(url_for('i_login'))
                
        # Handle profile picture upload
        if new_picture and new_picture.filename.strip():
            filename = secure_filename(new_picture.filename)
            file_path = os.path.join('static/uploads', filename)
            file_path = file_path.replace("\\", "/")
            new_picture.save(file_path)
        else:
            file_path = instructor[9]  # Keep old picture if no new one

        # Update the student's information in the database (except for student_id, phone_no, and email)
        cursor = mysql.connection.cursor()
        
        update_query = """
            UPDATE instructors
            SET name = %s, password = %s, course_name = %s, address = %s,
                date_of_birth = %s, blood_group = %s, profile_picture = %s
            WHERE instructor_id = %s
        """
        cursor.execute(update_query, (new_name, new_password, new_course_name, new_address, new_dob, new_blood_group, file_path, instructor_id))
        mysql.connection.commit()
        cursor.close()

        flash('Profile updated successfully!', 'success')
        return redirect(url_for('i_profile'))

    # Fetch instructor data from the database
    cursor.execute("SELECT * FROM instructors WHERE instructor_id = %s", (instructor_id,))
    instructor = cursor.fetchone()  # Fetch the instructor's record
    cursor.close()

    if instructor:  # Ensure instructor exists
        # Assign fetched data to variables
        instructor_id = instructor[1]  
        instructor_name = instructor[0]  
        phone_no = instructor[2]  
        email = instructor[3] 
        password = instructor[4] 
        course_name = instructor[5]  
        address = instructor[6]  
        date_of_birth = instructor[7] 
        blood_group = instructor[8]  
        profile_picture = instructor[9] 
        
        return render_template('i_profile.html', instructor=instructor, 
                               instructor_id=instructor_id, 
                               instructor_name=instructor_name, 
                               phone_no=phone_no,
                               email=email,
                               password=password,
                               course_name=course_name,
                               address=address,
                               date_of_birth=date_of_birth,
                               blood_group=blood_group,
                               profile_picture=profile_picture)
    
    else:
        flash("Instructor not found", 'danger')
        return redirect(url_for('i_login'))    
        
    # Render the profile template with instructor data
    return render_template('i_profile.html', instructor=instructor)


@app.route('/view_course/<int:course_code>')
def view_course(course_code):
    cursor = mysql.connection.cursor()

    # Fetch num_of_module from courses table
    cursor.execute("""
        SELECT num_of_module, num_of_students
        FROM courses
        WHERE course_code = %s
    """, (course_code,))
    course = cursor.fetchone()  # (num_of_module, num_of_students)

    if not course:
        cursor.close()
        return "Course not found", 404

    num_of_module, num_of_students = course

    # Calculate threshold
    value = (num_of_module * 10) + 20
    threshold = value * 0.8

    # Fetch all total_point from enrollment_status for the course
    cursor.execute("""
        SELECT student_id, total_point
        FROM enrollment_status
        WHERE course_code = %s
    """, (course_code,))
    enrollment_data = cursor.fetchall()  # List of tuples (student_id, total_point)

    # Update status based on threshold
    for student_id, total_point in enrollment_data:
        if total_point >= threshold:
            cursor.execute("""
                UPDATE enrollment_status
                SET status = 'completed'
                WHERE course_code = %s AND student_id = %s
            """, (course_code, student_id))
    mysql.connection.commit()

    # Count students with status='completed'
    cursor.execute("""
        SELECT COUNT(*)
        FROM enrollment_status
        WHERE course_code = %s AND status = 'completed'
    """, (course_code,))
    N = cursor.fetchone()[0]  # Number of students with 'completed' status

    # Calculate completion rate
    if num_of_students > 0:
        rate = round((N / num_of_students) * 100, 2)
    else:
        rate = 0.00

    # Update completion_rate in courses table
    cursor.execute("""
        UPDATE courses
        SET completation_rate = %s
        WHERE course_code = %s
    """, (rate, course_code))
    mysql.connection.commit()
    
    cursor.close()

    # Fetch course details from courses table
    cursor = mysql.connection.cursor()
    course_query = """
        SELECT course_name, category, num_of_students, completation_rate
        FROM courses
        WHERE course_code = %s
    """
    cursor.execute(course_query, (course_code,))
    course = cursor.fetchone()  # (course_name, category, num_of_students, completion_rate)

    if not course:
        cursor.close()
        return "Course not found", 404

    # Fetch content details from content table
    content_query = """
        SELECT module_no, outline_pdf, pdf_file, module_video, assignment, deadline
        FROM content
        WHERE course_code = %s
        ORDER BY module_no ASC
    """
    cursor.execute(content_query, (course_code,))
    content = cursor.fetchall()  # List of content for the course
    
    # Fetch feedback for the course (Students)
    feedback_query = """
        SELECT s.name, sf.feedback
        FROM student_feedback sf
        INNER JOIN students s ON sf.student_id = s.student_id
        WHERE sf.course_code = %s
        ORDER BY sf.f_no DESC
    """
    cursor.execute(feedback_query, (course_code,))
    feedbacks = cursor.fetchall()
    
    # Fetch feedback for the course (Instructors)
    ifeedback_query = """
        SELECT i.name, f.feedback
        FROM instructor_feedback f
        INNER JOIN instructors i ON f.instructor_id = i.instructor_id
        WHERE f.course_code = %s
        ORDER BY f.f_no DESC
    """
    cursor.execute(ifeedback_query, (course_code,))
    ifeedbacks = cursor.fetchall()
    
    cursor.close()

    return render_template(
        'i_view_course.html',
        course_code=course_code,
        course_name=course[0],
        course_category=course[1],
        num_of_students=course[2],
        completion_rate=course[3],
        content=content if content else [],  # Pass empty list if no modules
        feedbacks=feedbacks,
        ifeedbacks=ifeedbacks
    )
    
@app.route('/view_assignments/<int:course_code>')
def view_assignments(course_code):
    cursor = mysql.connection.cursor()

    # Fetch course name
    course_query = """
        SELECT course_name
        FROM courses
        WHERE course_code = %s
    """
    cursor.execute(course_query, (course_code,))
    course = cursor.fetchone()  # (course_name,)

    if not course:
        cursor.close()
        return "Course not found", 404

    # Fetch all assignments for the course
    assignments_query = """
        SELECT course_code, module_no, assignment, deadline
        FROM content
        WHERE course_code = %s
    """
    cursor.execute(assignments_query, (course_code,))
    all_assignments = cursor.fetchall()

    # Fetch submitted assignments for the course
    submitted_query = """
        SELECT a.student_id, s.name AS student_name, c.course_name, 
               a.module_no, a.submitted_assignments, a.submission_date, 
               a.marks
        FROM assignments a
        JOIN students s ON a.student_id = s.student_id
        JOIN courses c ON a.course_code = c.course_code
        WHERE a.course_code = %s
    """
    cursor.execute(submitted_query, (course_code,))
    submitted_assignments = cursor.fetchall()

    # Process Submission Status (on-time or late)
    processed_submitted = []
    for submission in submitted_assignments:
        submission_status = "On time"
        module_no = submission[3]
        submission_date = submission[5]

        # Fetch deadline
        cursor.execute("""
            SELECT deadline
            FROM content
            WHERE course_code = %s AND module_no = %s
        """, (course_code, module_no))
        deadline = cursor.fetchone()

        if submission_date > deadline[0]:
            submission_status = "Late"

        processed_submitted.append({
            "student_id": submission[0],
            "student_name": submission[1],
            "course_name": submission[2],
            "module_no": module_no,
            "submitted_assignments": submission[4],
            "submission_date": submission_date,
            "submission_status": submission_status,
            "marks": submission[6],
            "course_code": course_code
        })

    cursor.close()

    return render_template(
        'view_assignments.html',
        course_name=course[0],
        all_assignments=all_assignments,
        submitted_assignments=processed_submitted
    )

@app.route('/submit_marks/<int:student_id>/<int:course_code>/<int:module_no>', methods=['POST'])
def submit_marks(student_id, course_code, module_no):
    marks = request.form['marks']
    cursor = mysql.connection.cursor()
    update_query = """
        UPDATE assignments
        SET marks = %s
        WHERE student_id = %s AND course_code = %s AND module_no = %s
    """
    cursor.execute(update_query, (marks, student_id, course_code, module_no))
    cursor.execute("UPDATE enrollment_status SET total_point = total_point + %s WHERE course_code = %s AND student_id = %s", (marks, course_code, student_id))
    mysql.connection.commit()
    cursor.close()
    return redirect(f'/view_assignments/{course_code}')

    
@app.route('/submit_feedback/<int:course_code>', methods=['POST'])
def submit_feedback(course_code):
    if 'instructor_id' not in session:
        return redirect(url_for('i_login'))  # Redirect if the student is not logged in

    instructor_id = session['instructor_id']
    feedback = request.form['feedback']

    # Insert the feedback into the database
    cursor = mysql.connection.cursor()
    insert_query = """
        INSERT INTO instructor_feedback (course_code, instructor_id, feedback)
        VALUES (%s, %s, %s)
    """
    cursor.execute(insert_query, (course_code, instructor_id, feedback))
    mysql.connection.commit()
    cursor.close()

    flash("Your feedback has been submitted successfully.", "success")
    return redirect(url_for('view_course', course_code=course_code))

@app.route('/i_logout')
def i_logout():
    session.pop('instructor_name', None)  # Remove the instructor's name from the session
    return redirect(url_for('i_login'))

if __name__ == '__main__':
    app.run(debug=True, port=1717)
