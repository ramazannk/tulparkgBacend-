# tulparkgBacend-
app.post('/submit', upload.single('product'), async (req, res) => {
    const { lastName, productName, desc, car, password } = req.body;
    const imgBuffer = req.file.buffer;
        
    try {
        const hashedPassword = await bcrypt.hash(password, saltRounds);
        console.log("Hashed Password:", hashedPassword); 
        // Insert into database
        const result = await db.query(
            `INSERT INTO tulparkg (lastName, productName, description, car, img, password) VALUES ($1, $2, $3, $4, $5, $6) RETURNING id`, 
            [lastName, productName, desc, car, imgBuffer, hashedPassword]
        );

        res.json({ message: 'Posted successfully', id: result.rows[0].id });
    } catch (err) {
        console.log('Something went wrong: ' + err);
        res.status(500).json({ error: 'Failed to post image' });
    }
});




app.post('/register', async(req, res) => {
    const { name, password} = req.body;

    const hashedPassword = await bcrypt.hash(password, saltRounds);
        console.log("Hashed Password:", hashedPassword); 

        const query = `
        INSERT INTO admintkg (name, password) 
        VALUES ($1, $2) 
        RETURNING *;
    `;
    
    const values = [name, hashedPassword];  // The name and password values
    
    try {
        const result = await db.query(query, values);  // Execute the query
        console.log(result.rows[0]);  // Log the inserted row with the auto-generated ID
        res.json(result.rows[0]);  // Return the inserted row as a response
    } catch (err) {
        console.error('Error inserting data:', err);
        res.status(500).json({ error: 'Failed to insert data' });
    }
    
})

app.post('/login/admin', async (req, res) => {
    const { name, password} = req.body;
    console.log(name, password)

    try {
        const userResult = await db.query(
            `SELECT * FROM admintkg WHERE name = $1`,
            [name]
        );
        if (userResult.rows.length === 0) {
            return res.status(401).send('Invalid username or password');
        }

        const user = userResult.rows[0];

        const match = await bcrypt.compare(password, user.password);
        if(match){
            console.log(true)
        }

        if (!match) {
            console.log('no match')
            return res.status(401).send('Invalid username or password');
        }
        
        res.json("ok")
    } catch (err) {
        console.error(err);
        res.status(500).send('Error during login');
    }
});

app.post('/login', async (req, res) => {
    const { name, password} = req.body;
    console.log(name, password)

    try {
        const userResult = await db.query(
            `SELECT * FROM tulparkg WHERE lastname = $1`,
            [name]
        );

        if (userResult.rows.length === 0) {
            return res.status(401).send('Invalid username or password');
        }

        const user = userResult.rows[0];

        const match = await bcrypt.compare(password, user.password);
        if(match){
            console.log(true)
        }

        if (!match) {
            console.log('no match')
            return res.status(401).send('Invalid username or password');
        }


        const productsResult = await db.query(
            'SELECT * FROM tulparkg WHERE id = $1',
            [user.id]
        );

        const data = productsResult.rows.map((elem) => {
            let img = null;
            if (elem.img) {
                img = Buffer.from(elem.img).toString('base64');
            }
            return {
                id: elem.id,
                name: elem.lastname,
                productName: elem.productname,
                desc: elem.description,
                car: elem.car,
                img: img
            };
        });

        res.json(data[0]);
    } catch (err) {
        console.error(err);
        res.status(500).send('Error during login');
    }
});







  
app.get('/getMaps/:id', async(req, res) => {
    const { id } = req.params;

    try {
        const result = await db.query(`SELECT latitude, longitude FROM cars WHERE id = $1`, [id]);
        
        if (result.rows.length === 0) {
        return res.status(404).json({ error: 'Car not found' });
        }

        res.json(result.rows[0]);
    } catch (err) {
        console.error('Error fetching car location:', err);
        res.status(500).json({ error: 'Failed to fetch car location' });
    }
});



app.post('/submit/car', async (req, res) => {
    const { lat, lng, isConfurim, devices } = req.body;  // latitude and longitude from request body
    
    console.log(lat, lng, devices);  // Debugging output to see what's being sent

    const query = `
        INSERT INTO cars (latitude, longitude, updated_at, devices)
        VALUES ($1, $2, NOW(), $3)
        ON CONFLICT (id) 
        DO UPDATE SET 
        latitude = EXCLUDED.latitude, 
        longitude = EXCLUDED.longitude,
        updated_at = NOW(),
        devices = EXCLUDED.devices
        RETURNING *;
    `;

    const values = [parseFloat(lat), parseFloat(lng), devices];

    try {
        const result = await db.query(query, values);  // Executing the query
        if (result.rows.length > 0) {
            console.log('Car location updated:', result.rows[0]);  // Debugging the returned row
            res.json({ message: 'Car location updated successfully', car: result.rows[0] });
        } else {
            res.status(404).json({ message: 'Car not found' });  // Handle case if no row is returned
        }
    } catch (err) {
        console.error('Something went wrong:', err);
        res.status(500).json({ error: 'Failed to update car location' });  // Error handling
    }
});

  
  

app.delete('/delete/:id', async (req, res) => {
    console.log(req.params.id)
    const {id} = req.params;
    console.log(id)
    try {
        await db.connect();
        await db.query(`DELETE FROM tulparkg WHERE id = $1`, [id]); // Use dynamic ID
        console.log('Success');
        res.json({ message: 'Deleted successfully' });
    } catch (err) {
        console.log("Error: " + err);
        res.status(500).json({ error: 'Failed to delete image' });
    }
});


app.get('/product/:id', async(req, res) => {
    const{id} = req.params;
    try{
        await db.connect();
        const result = await db.query('SELECT * FROM tulparkg WHERE id = $1', [id]);
        if (result.rows.length === 0) {
            return res.status(404).json({ error: 'Product not found' });
        }
        const data = result.rows.map((elem) => {
            let img = null
            if (elem.img) {
                    img = Buffer.from(elem.img).toString('base64');
            }
            return{
                id: elem.id,
                name: elem.lastname,
                productName: elem.productname,
                desc: elem.description,
                car: elem.car,
                img: img
            }
        })
        res.json(data[0])

    }catch(err){
        console.log(err)
    }
    
});


app.get('/homepage', async(req, res) => {
    try{
        const result = await db.query('SELECT * FROM tulparkg');
        
        const data = result.rows.map(row => {
            const img = Buffer.from(row.img).toString('base64');
            return{
                id: row.id,
                name: row.lastname,
                productName: row.productname,
                desc: row.description,
                car: row.car,
                img: img
            }
        })
        res.json(data)
    }catch(err){
        console.log("Something went wrong" + err)
    }
});

app.get("/", (req, res) => {
    res.send("hello my api")
})

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
