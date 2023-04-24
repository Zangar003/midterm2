# midterm2
Zangar Tasbolat


Explain this code more specifically
package main
import (
	"database/sql"
	"fmt"
	"html/template"
	"net/http"

	"golang.org/x/crypto/bcrypt"

	"io/ioutil"
	"log"

	_ "github.com/go-sql-driver/mysql"
	"github.com/gorilla/sessions"
)

var store = sessions.NewCookieStore([]byte("mysession"))

var db *sql.DB

var err error

func CreateAconut(res http.ResponseWriter, req *http.Request) {
	db := dbConn()
	if req.Method != "POST" {
		http.ServeFile(res, req, "static/templates/signUP.html")
		return

	}

	username := req.FormValue("Name")
	email := req.FormValue("email")
	password := req.FormValue("Password")

	var user string
	err := db.QueryRow("SELECT Name FROM productdb.Products WHERE Name=?", username).Scan(&user)
	switch {
	case err == sql.ErrNoRows:
		hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
		if err != nil {
			http.Error(res, "Server error, unable to create your account.", 500)
			return
		}

		_, err = db.Exec("insert into productdb.Products (Name, Email, Password) values(?,?,?)", username, email, hashedPassword)

		if err != nil {
			http.Error(res, "Server error, unable to create your account.", 500)
			return
		}
		res.Write([]byte("User created!"))

		return

	case err != nil:
		http.Error(res, "Server error, unable to create your account.", 500)
		return
	default:
		http.Redirect(res, req, "/", 301)
	}
	defer db.Close()
}

func loginPage(res http.ResponseWriter, req *http.Request) {
	db := dbConn()
	if req.Method != "POST" {
		http.ServeFile(res, req, "static/templates/signIn.html")
		return
	}

	username := req.FormValue("username")
	password := req.FormValue("password")

	var databaseUsername string
	var databasePassword string

	err := db.QueryRow("SELECT Name, Password FROM  productdb.Products  WHERE Name=?", username).Scan(&databaseUsername, &databasePassword)

	if err != nil {
		http.Redirect(res, req, "/login", 301)
		return
	}

	err = bcrypt.CompareHashAndPassword([]byte(databasePassword), []byte(password))
	if err != nil {
		http.Redirect(res, req, "/login", 301)
		return
	}

	http.Redirect(res, req, "/", 301)
	res.Write([]byte("Hello  " + databaseUsername))
	defer db.Close()
}

func Logout(response http.ResponseWriter, request *http.Request) {
	session, _ := store.Get(request, "mysession")
	session.Options.MaxAge = -1
	session.Save(request, response)
	http.Redirect(response, request, "/loginIndex", http.StatusSeeOther)
}

func index(w http.ResponseWriter, r *http.Request) {

	t, err := template.ParseFiles("static/templates/index.html")
	if err != nil {
		fmt.Fprintf(w, err.Error())
	}

	t.Execute(w, "index")
}
func dbConn() (db *sql.DB) {

	db, err = sql.Open("mysql", "root:root@/productdb")
	if err != nil {
		panic(err.Error())
	}
	return db
}

type upfile struct {
	ID        int
	Fname     string
	Item_type string
	Star      sql.NullInt32
	Path      string
	Price     int
	Count     int
	Comment   sql.NullString
}

var tmpl = template.Must(template.ParseGlob("static/templates/*"))

func upload(w http.ResponseWriter, r *http.Request) {
	db := dbConn()
	var selDB *sql.Rows

	if r.Method == "POST" {
		rating := r.FormValue("raiting")
		price := r.FormValue("price")

		if rating == "raiting" {
			sel, err := db.Query("SELECT * FROM `upload` ORDER BY star ASC")
			selDB = sel
			if err != nil {
				panic(err.Error())
			}
		} else if price == "price" {
			sel, err := db.Query("SELECT * FROM `upload` ORDER BY price ASC")
			selDB = sel
			if err != nil {
				panic(err.Error())
			}
		} else {
			sel, err := db.Query("SELECT * FROM upload ORDER BY id DESC")
			selDB = sel
			if err != nil {
				panic(err.Error())
			}
		}
	} else {
		sel, err := db.Query("SELECT * FROM upload ORDER BY id DESC")
		selDB = sel
		if err != nil {
			panic(err.Error())
		}
	}

	upld := upfile{}
	res := []upfile{}
	for selDB.Next() {
		var id, price int
		var fname, item_type, path string
		var comment sql.NullString
		var star sql.NullInt32

		err = selDB.Scan(&id, &fname, &item_type, &star, &path, &price, &comment)
		if err != nil {
			panic(err.Error())
		}
		upld.ID = id
		upld.Fname = fname
		upld.Item_type = item_type
		upld.Star = star
		upld.Path = path
		upld.Price = price
		upld.Comment = comment
		res = append(res, upld)

	}

	upld.Count = len(res)

	if upld.Count > 0 {
		tmpl.ExecuteTemplate(w, "uploadfile.html", res)
	} else {
		tmpl.ExecuteTemplate(w, "uploadfile.html", nil)
	}

	db.Close()

}
func uploadFiles(w http.ResponseWriter, r *http.Request) {

	db := dbConn()
	if r.Method != "POST" {
		http.ServeFile(w, r, "static/templates/uploadfile.html")
		return
	}
	fname := r.FormValue("fname")
	item_type := r.FormValue("item_type")
	// star := r.FormValue("star")
	price := r.FormValue("price")

	r.ParseMultipartForm(200000)
	if r == nil {
		fmt.Fprintf(w, "No files can be selected\n")
	}

	formdata := r.MultipartForm
	fil := formdata.File["files"]
	for i := range fil {
		file, err := fil[i].Open()
		if err != nil {
			fmt.Fprintln(w, err)
			return
		}
		defer file.Close()

		tempFile, err := ioutil.TempFile("static/assets/uploadimage/", "upload-*.jpg")

		if err != nil {
			fmt.Println(err)
		}
		defer tempFile.Close()

		filepath := tempFile.Name()
		fileBytes, err := ioutil.ReadAll(file)
		if err != nil {
			fmt.Println(err)
		}

		tempFile.Write(fileBytes)

		insForm, err := db.Prepare("INSERT INTO upload(fname, item_type, path, price) VALUES(?,?,?,?)")
		if err != nil {
			panic(err.Error())
		} else {
			log.Println("data insert successfully . . .")
		}
		insForm.Exec(fname, item_type, filepath, price)

		log.Printf("Successfully Uploaded File\n")
		defer db.Close()

		http.Redirect(w, r, "/", 301)
	}

}
func delete(w http.ResponseWriter, r *http.Request) {
	db := dbConn()
	emp := r.URL.Query().Get("id")
	log.Println("deleted successfully", emp)
	delForm, err := db.Prepare("DELETE FROM upload WHERE id=?")
	if err != nil {
		panic(err.Error())
	}
	delForm.Exec(emp)
	log.Println("deleted successfully", emp)
	defer db.Close()
	http.Redirect(w, r, "/", 301)
}

func star(w http.ResponseWriter, r *http.Request) {
	db := dbConn()
	if r.Method != "POST" {
		http.ServeFile(w, r, "static/templates/uploadfile.html")
		return
	}
	star := r.FormValue("star")
	emp := r.FormValue("id")
	com := r.FormValue("comment")

	delForm, err := db.Prepare("UPDATE `upload` SET  star = ? , comment = ? WHERE id = ?")
	if err != nil {
		panic(err.Error())
	}
	delForm.Exec(star, com, emp)
	log.Println("Updated successfully", emp, star, com)

	defer db.Close()
	http.Redirect(w, r, "/", 301)
}
func Buy(w http.ResponseWriter, r *http.Request) {
	db := dbConn()
	emp := r.URL.Query().Get("id")
	delForm, err := db.Prepare("DELETE FROM upload WHERE id=?")
	if err != nil {
		panic(err.Error())
	}
	delForm.Exec(emp)
	log.Println("Updated successfully", emp)
	defer db.Close()
	http.Redirect(w, r, "/", 301)
}

// func show_korzina(w http.ResponseWriter, r *http.Request) {
// 	db := dbConn()
// 	selDB, err := db.Query("SELECT * FROM korzina ORDER BY id DESC")

// 	if err != nil {
// 		panic(err.Error())
// 	}
// 	upld := upfile{}
// 	res := []upfile{}
// 	for selDB.Next() {
// 		var id, price int
// 		var fname, item_type, path string
// 		var coment sql.NullString
// 		var star sql.NullInt32

// 		err = selDB.Scan(&id, &fname, &item_type, &star, &path, &price, &coment)
// 		if err != nil {
// 			panic(err.Error())
// 		}
// 		upld.ID = id
// 		upld.Fname = fname
// 		upld.Item_type = item_type
// 		upld.Star = star
// 		upld.Path = path
// 		upld.Price = price
// 		upld.Comment = coment
// 		res = append(res, upld)

// 	}

// 	upld.Count = len(res)

// 	if upld.Count > 0 {
// 		tmpl.ExecuteTemplate(w, "showboy.html", res)
// 	} else {
// 		tmpl.ExecuteTemplate(w, "showboy.html", nil)
// 	}

// 	db.Close()

// }
func handleRequest() {
	http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("./static"))))
	http.HandleFunc("/login", loginPage)
	http.HandleFunc("/signup", CreateAconut)
	http.HandleFunc("/logout", Logout)
	// http.HandleFunc("/", index)
	http.HandleFunc("/", upload)
	http.HandleFunc("/uploadfiles", uploadFiles)
	http.HandleFunc("/dele", delete)
	http.HandleFunc("/star", star)
	http.HandleFunc("/buy", Buy)
	// http.HandleFunc("/korzina", show_korzina)

	log.Println("Server started on: http://localhost:9000")

	http.ListenAndServe(":9000", nil)
}

func main() {
	handleRequest()

}

3 / 3


This code is a Go (Golang) program that provides a basic implementation of a web application with user authentication, account creation, and file uploading features. Here is a brief overview of the code:

The code imports the necessary packages, including the "database/sql" and "github.com/go-sql-driver/mysql" packages for database connectivity, the "html/template" package for rendering HTML templates, the "net/http" package for HTTP handling, and the "golang.org/x/crypto/bcrypt" package for password hashing.

The code initializes a new cookie store and a new database connection to a MySQL database.

The code defines several functions for handling various HTTP requests, including:

CreateAccount: This function handles requests for creating new user accounts. It checks if the request method is POST and then retrieves the form values for the username, email, and password. If the username is not already in the database, the password is hashed using bcrypt and then inserted into the database as a new user. If there is an error, the function returns an HTTP error response.

LoginPage: This function handles requests for the login page. It checks if the request method is POST and then retrieves the form values for the username and password. It then retrieves the corresponding user record from the database and checks if the password matches the hashed password in the database. If the password is correct, the function redirects the user to the home page; otherwise, it redirects the user back to the login page.

Logout: This function handles requests for logging out the user. It clears the session cookie and redirects the user back to the login page.

Index: This function handles requests for the home page. It parses an HTML template file and executes it, passing the "index" string as the data context.

Upload: This function handles requests for uploading files. It first checks if the request method is POST and then retrieves the form values for the rating and price. It then retrieves the corresponding file records from the database and sorts them based on the rating or price. Finally, it parses an HTML template file and executes it, passing the file records as the data context.

The code also defines a function for establishing a new database connection and a struct for storing file upload records.
